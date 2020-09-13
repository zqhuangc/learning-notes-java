### 位图（二维数组值为（0，1）boolean型）（对应位置有值则为1）
```Java
long[] words; 
// (每一个long代表64个值，每一个下标代表一个组)  
//取模
int wordIndex = wordIndex(bitIndex) //对64取模（wordIndex）
words[wordIndex] |= (1L << bitIndex);  //对64取余在跟之前的的值取或（理解：1左移几位就代表余的值多少）

//由模跟余可判断原值  
//wordIndex*64+words[wordIndex]=原值
```

