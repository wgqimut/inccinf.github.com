---
layout: post
title: "线程同步障barrier的简单实现（C pthread和python threading）"
description: ""
category: 
tags: [并行, C/C++, python]
---
{% include JB/setup %}

#什么是barrier
在使用多线程模型实现一些并行算法的时候，我们经常需要进行多个线程之间的同步，以保证各个线程能够以一致的步伐运转，不会因线程切换等原因导致有些线程快一步，而有些线程慢一步。

当然，并不是任何时候都需要进行线程同步，这是根据应用场景和算法而定的，只不过大部分并行算法的多线程实现都是需要同步的，比如prefix-sum, pointer jumping等。

线程同步的实现方法有很多种，常用的有信号量semaphore，条件变量condition，和这里说的同步障barrier。

barrier的用法非常简单，在任何需要线程同步的地方（如迭代的开始或结束）使用barrier_wait，各线程将阻塞在barrier处，直到所有线程都到达barrier后，所有线程被唤醒。

#Linux pthread barrier用法
在linux下，pthread库自带了barrier，可以直接使用，如下例子：   

	#include <stdio.h>
	#include <pthread.h>

	pthread_barrier_t barrier;

	void mythread(void *arg) {
		int n = (int)arg;
    	int i;
    	for(i = 0; i < 5; ++i) {
        	printf("Thread #%d: %d\n", n, i);
        	// 每次迭代完后都到达barrier
        	barrier_wait(&barrier);
    	}
	}

	int main() {
    	// 初始化barrier，需要同步的线程数为3
    	pthread_barrier_init(&barrier, NULL, 3);

    	pthread_t p1, p2, p3;
    	pthread_create(&p1, NULL, (void *)mythread, (void *)1);
    	pthread_create(&p2, NULL, (void *)mythread, (void *)2);
    	pthread_create(&p3, NULL, (void *)mythread, (void *)3);

    	pthread_join(p1, NULL);
    	pthread_join(p2, NULL);
    	pthread_join(p3, NULL);
    	return 0;
	}

运行结果类似:    
	
	Thread #1: 0
	Thread #2: 0
	Thread #3: 0
	Thread #1: 1
	Thread #2: 1
	Thread #3: 1
	Thread #1: 2
	Thread #2: 2
	Thread #3: 2
	Thread #1: 3
	Thread #2: 3
	Thread #3: 3
	Thread #1: 4
	Thread #2: 4
	Thread #3: 4
可以看出，三个线程确实是同步的，即每个线程的迭代步伐是一致的。

#barrier简单实现（C pthread）
虽然在linux下，barrier可以直接使用，但是在其他的一些系统中，也许并没有实现barrier，比如我使用的OS X，这时如果想使用barrier就需要自己实现一个，其实实现起来也非常简单，就是利用了pthread的condition条件变量配合锁来实现。

pthread condition的作用就是将线程阻塞以等待一个条件的成立，一旦条件成立，某个线程就发出唤醒信号，使得阻塞的线程继续运行，使用它再加上等待计数，即可实现一个简单的barrier。代码如下：    
`barrier.h`

	/*
	 * A simple barrier implemention
	 */
	
	#ifndef BARRIER_H
	#define BARRIER_H
	
	#include <pthread.h>
	
	typedef struct {
	    pthread_mutex_t mutex;
	    pthread_cond_t cond;
	    unsigned int needed;
	    unsigned int called;
	}barrier_t;
	
	int barrier_init(barrier_t *b, unsigned int needed);
	int barrier_destroy(barrier_t *b);
	int barrier_wait(barrier_t *b);
	
	#endif
`barrier.c`
	
	/*
	 * A simple barrier implemention
	 */
	
	#include <stdlib.h>
	#include "barrier.h"
	
	int barrier_init(barrier_t *barrier, unsigned int needed)
	{
	    pthread_mutex_init(&barrier->mutex, NULL);
	    pthread_cond_init(&barrier->cond, NULL);
	    barrier->needed = needed;
	    barrier->called = 0;
	    return 0;
	}
	
	int barrier_destroy(barrier_t *barrier)
	{
	    pthread_mutex_destroy(&barrier->mutex);
	    pthread_cond_destroy(&barrier->cond);
	    return 0;
	}
	
	int barrier_wait(barrier_t *barrier)
	{
	    pthread_mutex_lock(&barrier->mutex);
	    barrier->called++;
	    if (barrier->called == barrier->needed) {
	        barrier->called = 0;
	        pthread_cond_broadcast(&barrier->cond);
	    } else {
	        pthread_cond_wait(&barrier->cond,&barrier->mutex);
	    }
	    pthread_mutex_unlock(&barrier->mutex);
	    return 0;
	}

#barrier简单实现（python）
仿照C语言的版本，同样可以很容易地实现一个python版本，也是使用了python threading的condition来实现，代码如下：

	import threading
	
	# barrier
	con = threading.Condition()
	called = 0
	needed = 0
	
	def barrier_init(k):
	    global needed
	    needed = k
	
	def barrier_wait():
	    global con
	    global called
	    global needed
	    con.acquire()
	    called += 1
	    if (called == needed or needed == 0):
	        called = 0
	        con.notifyAll()
	    else:
	        con.wait()
	    con.release()
	   
#小结
使用barrier可以简化并行算法的同步实现方法，比起信号量等更加容易理解和使用，同时barrier的实现也非常简单，可以根据自己的需要定制专用的barrier，比如增加一个barrier_exit和barrier_enter函数，来实现同步线程数目的动态变化。

