- [为什么Scan的速度比Get快？](#为什么scan的速度比get快)

# 为什么Scan的速度比Get快？
因为找寻key的时候会有大量的计算开销，包括以下这些：
* bloom过滤器
* 虚函数调用
* key比较
* IO

而scan可以节省大量的CPU cache miss开销以及虚函数调用开销。同时还可以并行IO。