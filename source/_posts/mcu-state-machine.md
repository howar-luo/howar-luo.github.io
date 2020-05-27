---
title: 基于baremetal的状态机设计
date: 2020-01-15 22:03:03
categories:
tags:
---

## 前言
单片机程序开发通常直接基于baremetal，或者使用较为简单的RTOS，虽然单片机开发程序相对来说较为简单，但同时也少了一些辅助软件系统设计的工具，***一不小心***就使得代码看起来杂乱无章，虽然功能都能实现，但对后期维护通常都是一种灾难。这篇blog结合实际工作中的总结，试图给出一种基于baremetal（基于RTOS/Linux当然也可以用）的状态机设计，使代码开发和维护都变得简单，也使代码更优雅。

## 状态机设计
本状态机设计方案中，共有4个元素：
* 状态
* 事件
* 状态机
* 调度器

### 状态
首先，给出状态的定义如下：
```C
typedef struct _state {
    int event;  /* 触发该状态流转的事件，下面会进行介绍 */
    state_handle_func_t handle;  /* 该状态的动作函数，即该状态被上述事件触发后执行该函数 */
    int16_t success_action;  /* 上述动作函数执行成功后，该状态的跳转动作（通常为跳转到下一个状态） */
    int16_t wait_action;  /* 上述动作函数执行后需等待，该状态的跳转动作（通常为仍然执行该状态） */
    int16_t fail_action;  /* 上述动作函数执行失败后，该状态的跳转动作（通常为终止该状态机） */
} state_t;
```
其中，状态的动作函数定义如下：
```C
typedef int (*state_handle_func_t)(void* context, void* feedback);
```
将上述多个状态放入状态机中，即可实现包含若干个状态的状态机。

### 事件
状态机中状态在两种情况下可进行流转：周期性的调度器；特定事件的发生。这两种情况都离不开事件。在本设计中，定义了以下事件：
```C
#define STATE_MACHINE_EVENT_ALL     (0x00) /* 该事件通常保留 */
#define STATE_MACHINE_EVENT_MAIN    (0x01) /* 周期性调度器会使用该事件进行状态的执行和跳转 */
#define STATE_MACHINE_EVENT_TIMEOUT (0x02) /* 周期性调度器会使用该事件触发超时状态的执行 */
#define STATE_MACHINE_EVENT_TIMER   (0x04) /* 周期性调度器会使用该事件触发状态周期性timer函数的执行 */
```
当然实际工程中还可以添加其他事件，比如某状态是传感器数据触发的，那么可以添加如下状态（下文会介绍具体如何使用）：
```C
#define STATE_MACHINE_EVENT_SENSOR  (0x08)
```

### 状态机
同样，首先看看状态机定义如下：
```C
typedef struct _state_machine {
    state_machine_timer_handle_func_t timer_handle;  /* 若不为NULL，则会被定时性调用 */
    state_machine_callback_t callback;  /* 若不为NULL，则会在该状态机结束时调用 */
    int index;  /* 当前状态索引 */
    int size;  /* 该状态机所包含的状态个数 */
	int canceled;  /* 该状态机是否被取消 */
    state_t states[1];  /* 该状态机所包含的状态，这里定义为长度为1的数组只是编程技巧，实际可包含若干个状态 */
} state_machine_t;
```
对于状态机而言，最重要的是：
* 创建状态机
* 添加状态
* 状态跳转

下面一一分析。
#### 创建状态机
代码如下：
```C
state_machine_t* state_machine_create(
    uint16_t state_count, uint16_t context_size, void** context) {
    size_t size = sizeof(state_machine_t)
                  + sizeof(state_t) * (state_count - 1) + context_size;
    state_machine_t* sm = (state_machine_t*)malloc(size);
    memset(sm, 0, size);
    
    if (context) {
        *context = (uint8_t*)sm + size - context_size;
    }
    
    return sm;
}
```
如上，创建状态机使用到了动态内存分配，所以在状态机结束时需要释放对应内存。在这个接口设计中，总共三个参数：
* state_count: 这个很好理解，就是状态的个数
* context_size: 为每个状态机申请了一段内存，里面存放该状态机运行的上下文context，其里面的内容可以自定义。该参数就是context长度
* context: 指向该上下文内存首地址的指针

#### 添加状态
代码如下：
```C
void state_machine_add(
    state_machine_t* state_machine,
    int event, 
    state_handle_func_t handle, 
    int16_t success_action,
    int16_t wait_action,
    int16_t fail_action) {
    state_t* state = &state_machine->states[state_machine->size++];
    state->event = event;
    state->handle = handle;
    state->success_action = success_action;
    state->wait_action = wait_action;
    state->fail_action = fail_action;
}
```
该函数比较简单，不作过多解释。实际工程中可能会大量调用该接口，用宏对其封装如下：
```C
#define STATE_MACHINE_EVENT2(e1, e2) \
    (STATE_MACHINE_EVENT_##e1 | STATE_MACHINE_EVENT_##e2)
#define STATE_MACHINE_EVENT3(e1, e2, e3) \
	(STATE_MACHINE_EVENT2(e1, e2) | STATE_MACHINE_EVENT_##e3)

#define STATE_MACHINE_ADD_EXT(sm, event, handle, success, wait, error) \
    state_machine_add( \
        sm,  \
        STATE_MACHINE_##event, \
        (state_handle_func_t)handle, \
        STATE_MACHINE_ACTION_##success, \
        STATE_MACHINE_ACTION_##wait, \
        STATE_MACHINE_ACTION_##error \
    )
	
#define STATE_MACHINE_ADD(sm, event, handle, success, wait, error) \
    state_machine_add( \
        sm,  \
        STATE_MACHINE_EVENT_##event, \
        (state_handle_func_t)handle, \
        STATE_MACHINE_ACTION_##success, \
        STATE_MACHINE_ACTION_##wait, \
        STATE_MACHINE_ACTION_##error \
    )
```
添加单事件触发的状态，使用`STATE_MACHINE_ADD`就可以了；有些状态的触发，可能多个事件中的任何一个就可以，那么可以使用`STATE_MACHINE_ADD_EXT`来添加：
```C
/* 添加单事件触发示例 */
STATE_MACHINE_ADD(sm, MAIN, state_handler, NEXT, NEXT, BREAK);
/* 添加多事件触发示例 */
STATE_MACHINE_ADD_E(sm, EVENT2(MAIN, SENSOR), state_handler, NEXT, RETRY, BREAK);
```

#### 状态跳转
状态跳转函数是状态机的核心，其代码如下：
```C
static int state_machine_run(
    int event, void* context, state_machine_t* state_machine, void* feedback) {
    if (state_machine->canceled || (state_machine->index >= state_machine->size)) {
        return state_machine->callback(state_machine, 0, context);
    }
    
    state_t* state = &state_machine->states[state_machine->index];    
    switch (event) {
        case STATE_MACHINE_EVENT_SENSOR:
        case STATE_MACHINE_EVENT_MAIN: {
            if (state->event != 0 && (state->event & event) != event) {
                return SCHEDULE_CONTINUE;
            }
        
            /* Run current state handle function */
            int16_t action = 0;
            int ret = state->handle(context, feedback);
            
            /* Return code convert to state machine action */
            switch (ret) {
                case STATE_MACHINE_OK:
                    action = state->success_action;
                    break;
                case STATE_MACHINE_WAIT:
                    action = state->wait_action;
                    break;
                case STATE_MACHINE_ERROR:
                    action = state->fail_action;
                    break;
                default:
                    action = STATE_MACHINE_ACTION_BREAK;
                    break;
            }
            
            /* Handle status action */
            switch (action) {
                case STATE_MACHINE_ACTION_NEXT:                
                    state_machine->index++;
                case STATE_MACHINE_ACTION_RETRY:
                    return SCHEDULE_CONTINUE;
                case STATE_MACHINE_ACTION_BREAK:
                    return state_machine->callback(state_machine, ret, context);
                default:
                    state_machine->index += action;
                    return SCHEDULE_CONTINUE;
            }
        }
        case STATE_MACHINE_EVENT_TIMER: {
            state_machine->timer_handle(context);
            return SCHEDULE_CONTINUE;
        }
        case STATE_MACHINE_EVENT_TIMEOUT: {
            return state_machine->callback(
				state_machine, STATE_MACHINE_TIMEOUT, context);
        }
    }
    
    return SCHEDULE_CONTINUE;
}
```
对于该函数，重点作以下说明：
* 该函数的返回值有两个：`SCHEDULE_CONTINUE`和`SCHEDULE_OK`，当返回`SCHEDULE_OK`时表明这个状态机已经执行完毕
* 状态的跳转不是仅仅可以顺序向下跳转，也可以往回跳，也可以跨状态跳转，只要在添加状态时使用合适的`action`参数即可，比如：
```C
#define STATE_MACHINE_ACTION_NEXT  (0x7FF0)
#define STATE_MACHINE_ACTION_RETRY (0x7FF1)
#define STATE_MACHINE_ACTION_BREAK (0x7FF2)

//Forward
#define STATE_MACHINE_ACTION_FORWARD_1  (1)
#define STATE_MACHINE_ACTION_FORWARD_2  (2)
#define STATE_MACHINE_ACTION_FORWARD_3  (3)
...
//Backward
#define STATE_MACHINE_ACTION_BACKWARD_1  (-1)
#define STATE_MACHINE_ACTION_BACKWARD_2  (-2)
#define STATE_MACHINE_ACTION_BACKWARD_3  (-3)
...
```
### 调度器
既然叫调度器，那么它处理的便是任务；说是任务，但其实和传统意义上的操作系统的任务还是有点区别的，因为这里的任务主要是包含状态机及状态机相关数据的集合，其定义如下：
```C
typedef struct _schedule_task {
    schedule_handle_func_t handle;  /* 任务（状态机）被调度到时执行的函数 */
    void* context;  /* 任务（状态机）的上下文 */
    void* sm;  /* 任务对应的状态机 */
    void* feedback;  /* 任务的反馈输入（比如收到的传感器数据） */
	uint32_t active_time;  /* 任务（状态机）被激活的时间戳 */
    uint32_t timeout;  /* 任务（状态机）超时时间 */
    uint32_t recently_wakeup_time;  /* 任务（状态机）最近一次被调度的时间戳 */
    uint32_t timer;  /* 任务（状态机）定时时间周期 */
} schedule_task_t;
```
其中`schedule_handle_func_t`的定义如下：
```C
typedef int (*schedule_handle_func_t)(
    int event, void* context, void* callback, void* feedback);
```
实际上对应这上文中的函数：
```C
static int state_machine_run(
    int event, void* context, state_machine_t* state_machine, void* feedback)
```
对于调度器而言，最重要的是提供两方面的能力：任务的添加、任务的调度执行。分别介绍如下：
* 任务的添加

首先，需要一个ring buffer来存储管理任务队列，其数据结构如下：
```C
#define SCHEDULE_QUEUE_SIZE (5)
typedef struct _schedule_task_queue {
    schedule_task_t task[SCHEDULE_QUEUE_SIZE];
    uint32_t first;
    uint32_t last;
} schedule_task_queue_t;

typedef struct _schedule {
    schedule_task_queue_t queue;
} schedule_t;

static schedule_t s_schedule;
```
新的任务可以从头部，也可以从尾部添加进该任务队列。需要注意的是，对于一个任务队列而言，其任务执行是阻塞机制的，即只有队首的任务执行完（状态机的所有状态都执行完）之后，接下来的任务才会执行，这样可以把相关联的状态机放入同一个队列中。如果有需要，可以创建多个队列（即多个`schedule_t`类型的变量）。

介绍完上述背景后，就可以给出任务添加的核心接口了，即获取队列中相应的任务指针（获取到后，进行相应的赋值即可）：
```C
/* 将任务放置到队首 */
static schedule_task_t* schedule_task_queue_push_front(schedule_task_queue_t* queue) {
    uint32_t prev = (queue->first + SCHEDULE_QUEUE_SIZE - 1) % SCHEDULE_QUEUE_SIZE;
    if (prev == queue->last) {
        return NULL;
    }
    schedule_task_t* result = &queue->task[prev];
    queue->first = prev;
    return result;
}
/* 将任务放置到队尾 */
static schedule_task_t* schedule_task_queue_push_back(schedule_task_queue_t* queue) {
    uint32_t next = (queue->last + 1) % SCHEDULE_QUEUE_SIZE;
    if (next == queue->first) {
        return NULL;
    }
    schedule_task_t* result = &queue->task[queue->last];
    queue->last = next;
    return result;
}
```
* 任务的调度执行

任务的调度执行主要完成以下工作：
1. 总是执行处于队首的状态机任务，当队首状态机任务全部执行完毕后，负责将队首的位置移到下一个状态机任务；
2. 检查当前队首状态机任务的周期性timer是否到达，若到达则执行相关函数；
3. 检查当前队首状态机任务是否超时，若超时，则负责将队首的位置移到下一个状态机任务。

对应的代码实现如下：
```C
static schedule_task_t* schedule_task_queue_peek(schedule_task_queue_t* queue) {
    return queue->first != queue->last ? &queue->task[queue->first] : NULL;
}
static schedule_task_t* schedule_task_queue_pop(schedule_task_queue_t* queue) {
    if (queue->first == queue->last) {
        return NULL;
    }
    
    schedule_task_t* first = schedule_task_queue_peek(queue);
    queue->first = (queue->first + 1) % SCHEDULE_QUEUE_SIZE;

    return first;
}
static void schedule_move_next(void) {
	schedule_task_t* task = NULL;
	
	schedule_task_queue_pop(&s_schedule.queue);
			
	if ((task = schedule_task_queue_peek(&s_schedule.queue)) != NULL) {
        /* get_tick_msec()为获取当前系统时间戳接口，与具体平台相关 */
		task->active_time = get_tick_msec();
	}
}
void schedule_handle(int event) {
    /* get_tick_msec()为获取当前系统时间戳接口，与具体平台相关 */
    uint32_t now = get_tick_msec();
    schedule_task_t* task = NULL;

    //Handle queue's task
    while ((task = schedule_task_queue_peek(&s_schedule.queue)) != NULL) {
        if ((task->active_time + task->timeout) <= now) {
            task->handle(
                STATE_MACHINE_EVENT_TIMEOUT, task->context, task->callback, NULL);
            schedule_move_next();
            
            continue;
        } 

        //Check timer and wakeup handle function for timer event
        if ((task->timer > 0) && (now - task->recently_wakeup_time >= task->timer)) {
            task->handle(STATE_MACHINE_EVENT_TIMER, task->context, task->callback, NULL);
            //Update wake up time
            task->recently_wakeup_time = get_tick_msec();
        }
        
        if (task->handle(event, task->context, task->callback, NULL) == SCHEDULE_OK) {
            schedule_move_next();
            
            continue;
        } 
        
        break;
    }
}
```
`schedule_handle`需要在主函数中周期性调用，且通常传入参数`STATE_MACHINE_EVENT_MAIN`。

## 小结
通过对上述元素的实现，可以得到一个可用的（也算好用的），能在单片机系统（也可在RTOS/Linux上）运行的状态机。

