> [主頁](https://hackmd.io/@8thEdition) > [深入Python記憶體回收機制(Garbage Collection)](https://hackmd.io/@8thEdition/garbage_collection)
# 深入 Python 記憶體回收機制 (Garbage Collection)


寫Python寫了好一陣子，開始對Python底層運作開始感興趣，因此最近研究了一下最多人使用的CPython的垃圾回收機制，在這邊對自己目前了解到的東西做了一下筆記。接下來將針對三部分逐一說明。

1. [(主)引用計數機制](https://hackmd.io/@8thEdition/cpython_garbage_collection_reference_counting)
2. (輔)標記清除機制 (還在努力中...)
3. (輔)GC分代機制 ）(還在努力中...)


> [name=8thEdition]本文所有的程式碼執行結果皆是直接執行.py所產生，且為Python v3.10的版本，若發現結果不一致，有可能是使用互動式介面的關係喔！
