Naive 顺序读写混合+校验
```shell
rw="rw" # 读写模式: rw randrw read write randread randwrite trim randtrim
size="100%" # 写入大小: 1G/20%
runtime="1200" # 执行时间
```
![[Pasted image 20241208161649.png]]
Naive 随机读写混合+校验
![[Pasted image 20241208164227.png]]

这次不进行数据校验就触发了GC
![[Pasted image 20241208170735.png]]
GC稳定后：
![[Pasted image 20241208172606.png]]
可能是去掉边写边擦，随机写：
![[Pasted image 20241208221849.png]]
去掉边写边擦，随机写：
![[Pasted image 20241209004308.png]]
去掉边写边擦，顺序写：
![[1733711843796.png]]

测试trim命令：
![[Pasted image 20241209130059.png]]



**等待对比**
![[Pasted image 20241218112706.png]]![[Pasted image 20241218122505.png]]

![[Pasted image 20241218213604.png]]