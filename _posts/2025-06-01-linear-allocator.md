---
title: "内存管理：线性分配器"
categories:
  - C
tags:
  - Memory Management
---

线性分配器（Linear Allocator），也常被称为区域分配器（Arena Allocator）或顺序分配器（Sequential Allocator），是一种简单高效的内存管理策略。它特别适用于内存需求具有清晰生命周期边界的场景，即“分步创建，一同销毁”的线性任务。

# 示例1：Web请求处理

Web服务器对HTTP请求的处理通常遵循一系列线性步骤，例如：

```
接收请求 (Receive)
  => 解析请求 (Parse)
    => 权限验证 (Validate)
      => 业务处理 (Process)
        => 构建响应 (Generate)
          => 发送响应 (Send)
```

示例代码如下：

```c
int OnHttpRequest(char* request)
{
	// 创建并初始化一个线性内存分配器
	LinearAllocatorContext _ctx; // 在当前栈上分配上下文
	LinearAllocatorContext* ctx = &_ctx; // 使用指针
	LinearAllocator_Initialize(ctx, 10 * 1024 * 1024); // 预分配10MB

	// “分步创建”：

	// 步骤1：解析请求
	char* line_buffer;
	char* header_buffer;
	char* body_buffer;

	LineParser_Parse(ctx, request, &line_buffer); 	// 所有子任务中的动态内存都从线性分配器 ctx 中获取
	HeaderParser_Parse(ctx, request, &header_buffer); // 各种Parser都使用ctx来分配内部的动态数据（如子串、子节点等）
	BodyParser_Parse(ctx, request, &body_buffer);

	// 步骤2：验证请求
	CheckURL(ctx, line_buffer); // 在验证函数中需要临时使用的内存，从 ctx 分配
	CheckAuth(ctx, header_buffer);

	// 步骤3：处理请求
	JSON json_body;
	JSON_Parse(ctx, body_buffer, &json_body); // 对JSON的处理过程中需要进行动态内存分配，使用ctx

	int status_code;
	JSON json_response;
	Dispatch(ctx, &json_body, &status_code, &json_response); // 业务处理过程中需要进行动态内存分配，从ctx分配

	// 步骤4：生成响应
	char* response_header;
	char* response_body;

	MakeResponseHeader(ctx, status_code, &response_header); // MakeResponseHeader内部使用ctx分配
	MakeResponseBody(ctx, &json_response, &response_body);  // MakeResponseBody内部使用ctx分配

	// 步骤5：发送响应
	ResolveAndSend(response_header, response_body);

	// “一同销毁”：
	// 请求处理完毕，释放线性分配器及其分配的所有内存
	LinearAllocator_Free(ctx);
	return 0;
}

int main(int argc, char* argv[])
{
	WebServer_Listen(8080, OnHttpRequest);
	return 0;
}
```

每一步都会依赖前面步骤产生的结果，在每一步中我们都使用同一个线性内存分配器来分配新的内存空间，当这次请求处理完成之后，我们通过直接释放这个分配器实现对内存空间的一同销毁

在线性内存分配器内部，它会在初始化时直接请求系统（malloc）一大块内存空间用于分配，每次分配后不做垃圾回收，只是简单的向后移动，直到任务完成，进行整体释放

这样的模式有助于减少系统分配（malloc）的次数从而提高性能，同时也有助于减少内存的碎片化

# 示例2：游戏引擎中生成帧

游戏引擎以非常高的频率渲染画面（例如，30fps、60fps或更高）。每一帧的生成都涉及到大量的内存操作和计算：

- 更新游戏对象状态（位置、大小、方向）
- 更新对象可见性（哪些对象在当前视野内）
- 构建渲染队列
- 处理光照、碰撞、粒子效果等

这些过程需要大量的动态内存分配来存储这一帧的临时数据，但因为这些数据通常只在该帧有效，一旦帧渲染完成，它们就可以丢掉了。线性分配器非常适合这种“帧级”内存管理

```c
void GameLoop_RenderFrame() {
	// 每帧开始时，重置帧分配器
	// 直接忽略上一帧时在分配中分配的所有内存，在这一桢从头开始使用，速度会非常快
	LinearAllocator_Reset(Get_LinearAllocatorContext());

	UpdateGameObjects(...); // 更新游戏逻辑(对象状态等)
	BuildRenderQueue(...); // 构建渲染队列
	RenderToGPU(...);
}

int main(int argc, char* argv[]) {
	LinearAllocatorContext _ctx;
	LinearAllocatorContext* ctx = &_ctx;

	GameEngine_Initialize({ ctx, ... }); // 初始化各种资源，游戏引擎会将分配器上下文放在全局

	while (1) {
		GameLoop_RenderFrame();
		...
	}

	GameEngine_Free({ ctx, ... });
	return 0;
}
```

# 参考实现

根据上面两个示例，线性分配器提供的功能其实很简单，我们用下面一组函数来描述：

```c
// 初始化分配器，在开始使用之前调用
int LinearAllocator_Initialize(LinearAllocatorContext* ctx, size_t length);
// 重置，清空分配器，用于快速重用
int LinearAllocator_Reset(LinearAllocatorContext* ctx);
// 释放内部内存，在使用完毕后调用
int LinearAllocator_Free(LinearAllocatorContext* ctx);

// 申请分配
int LinearAllocator_Alloc(LinearAllocatorContext* ctx, size_t size, void** target);
```

要实现一个线性分配器，需要知道我们在哪里为它开辟了多少内存，然后维护一个已经分配走的空间的指针，分配一次，指针就向后移动一次，可以用类似下面这样的数据结构来描述

```c
// 线性分配器上下文
typedef struct LinearAllocatorContext {
	unsigned char* head; // 起始地址
	size_t length;       // 总大小
	size_t offset;       // 当前偏移量
} LinearAllocatorContext;
```

接口实现

```c
// 初始化线性分配器
// 参数:
//   ctx: 分配器上下文
//   length: 需要分配的内存块大小（字节）
// 返回值:
//   0: 成功
//   1: 无效的分配器上下文
//   2: 内存分配失败
int LinearAllocator_Initialize(LinearAllocatorContext* ctx, size_t length)
{
	if (ctx == NULL)
	{
		return 1;
	}

	void* head = malloc(length);
	if (head == NULL)
	{
		return 2;
	}

	ctx->head = (unsigned char*)head;
	ctx->length = length;
	ctx->offset = 0;

	return 0;
}

// 从分配器中申请内存
// 参数:
//   ctx: 分配器上下文
//   size: 请求的内存大小（字节）
//   target: 返回分配的内存地址
// 返回值:
//   0: 成功
//   1: 无效的分配器上下文
//   2: 无效的请求大小
//   3: 分配器空间不足
int LinearAllocator_Alloc(LinearAllocatorContext* ctx, size_t size, void** target)
{
	if (ctx == NULL)
	{
		return 1;
	}

	if (size <= 0)
	{
		return 2;
	}

	if (ctx->offset + size > ctx->length)
	{
		return 3;
	}

	*target = ctx->head + ctx->offset;
	ctx->offset += size; // 向后移动

	return 0;
}

// 重置分配器（释放所有已分配内存，保留内存块）
// 参数:
//   ctx: 分配器上下文
// 返回值:
//   0: 成功
//   1: 无效的分配器上下文
int LinearAllocator_Reset(LinearAllocatorContext* ctx)
{
	if (ctx == NULL)
	{
		return 1;
	}

	ctx->offset = 0; // 重置偏移量

	return 0;
}

// 释放分配器（释放所有已分配内存）
// 参数:
//   ctx: 分配器上下文
// 返回值:
//   0: 成功
//   1: 无效的分配器上下文
int LinearAllocator_Free(LinearAllocatorContext* ctx)
{
	if (ctx == NULL)
	{
		return 1;
	}

	free(ctx->head);

	ctx->head = NULL;
	ctx->length = 0;
	ctx->offset = 0;

	return 0;
}
```
