
```
pv1与pvc1通过storageClassName，指定绑定
pv2与pvc2通过storageClassName，指定绑定，pvc中meta中lable 不特别调用可以忽略,如果指定了label，标签必须匹配，否则不能创建成功
pv3与pv4 无storageClassName,pvc3 随机调用pv3,pv4

```
