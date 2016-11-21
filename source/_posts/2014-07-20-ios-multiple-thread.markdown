layout: 

title: "iOS Multiple Thread"

date: 2014-07-20 10:54:29 +0800

comments: true

categories: 技术

tags: Other

---

# 1. 使用POSIX创建多线程

(1). 创建出pthread_attr_t，并设置属性
(2). 调用pthread_create创建出线程，此时已经开始运行
```
void* PosixThreadMainRoutine(void* data)
{    
    // Do some work here.    
    return NULL;
}
void LaunchThread()
{    
    // Create the thread using POSIX routines.    
    pthread_attr_t  attr;    
    pthread_t       posixThreadID;    
    int             returnVal;    
    returnVal = pthread_attr_init(&attr);    
    assert(!returnVal);    
    returnVal = pthread_attr_setdetachstate(&attr, THREAD_CREATE_DETACHED);    assert(!returnVal);    
    int     threadError = pthread_create(&posixThreadID, &attr, &PosixThreadMainRoutine, NULL);    
    returnVal = pthread_attr_destroy(&attr);    
    assert(!returnVal);    
    if (threadError != 0)    
    {        
        // Report an error.    
    }
}
```