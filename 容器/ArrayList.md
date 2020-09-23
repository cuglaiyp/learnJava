# ArrayList

1. ArrayList底层由Object[] elementData数组实现，其中批量删除、添加多由Arrays.copy()方法实现，而Arrays.copy()又调用的System.arraycopy()，所以自己复制数组的时候可以使用System.arraycopy()方法，速度较快。