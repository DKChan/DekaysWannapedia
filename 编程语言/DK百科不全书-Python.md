# **经验笔记**

## **调试**

### **性能排查工具**

#### **CPU/执行时间**

##### **cProfile + Snakevit**

太卡了, 无法扫描async函数的调用; 垃圾

##### **py-spy**

类似pprof, 生成火焰图, svg格式文件可以直接浏览器查看

简易用法

##### **pyinstrument** 

async调用的需要用这个工具, 原始工具计算不准确, 这里会将协程调用加回去

直接打开生成的html,输出界面很棒,可以看到每次调用的堆栈,渲染也不卡

## **组件**

### **Web**

#### **Uvicorn**

ASGI服务器, FastAPI的组件

##### **多进程启动**

<https://uvicorn.dev/#running-with-gunicorn>

官方推荐使用gunicorn开启多进程

而不是使用uvicorn workers(当前已经deprecated), 有独立的新包uvicorn-worker

但生产环境部署还是推荐gunicorn, 因为他的进程管理等能力
