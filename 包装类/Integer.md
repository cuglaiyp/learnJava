## Integer

## getChars方法

~~~java
//将给定的int i原样转换成字符串
static void getChars(int i, int index, char[] buf) {
    int q, r;
    int charPos = index;
    char sign = 0;
    
    if (i < 0) {
        sign = '-';
        i = -i;
    }

    // 如果i >= 65536， 就一次性取末尾两位转换
    while (i >= 65536) { //假设此时i = 65536
        q = i / 100; 	// q = 655	
        //(q << 6) + (q << 5) + (q << 2) = q *(2的6次+2的5次+2的2次) = q*100 = 65500
        r = i - ((q << 6) + (q << 5) + (q << 2)); // r = 65536 - 65500 = 36
        i = q;
        buf [--charPos] = DigitOnes[r];
        buf [--charPos] = DigitTens[r];
    }

    // 如果i < 65536，就每次取末尾一位转换
    for (;;) {//假设此时 i = 65535
        // 向右无符号右移19位，即除以2的19次方（524288），所以前面乘以524288/10 + 1 = 52429
        //（加1是为了避免向下取整而产生误差）,即得到q ≈ i/10 = 6553
       	// 那为什么是无符号右移19位不是18位、20位呢？ 
        
        // 2^10=1024, 103/1024=0.1005859375
		// 2^11=2048, 205/2048=0.10009765625
		// 2^15=32768, 3277/32768=0.100006103515625
		// 2^16=65536, 6554/65536=0.100006103515625
		// 2^19=524288, 52429/524288=0.10000038146972656
		// 2^20=1048576, 104858/1048576=0.1000003815
        
        // 这里有两个前提——效率：移位 > 乘法 > 除法
        
        // 可以看到无符号右移位数越多，精度越高，所以右移位数尽量多，但是因为int型只有32个二进制位
        // 右移位数越多，i乘以的数就越大，为了不溢出，i就只能尽量小。i小了之后，更多的数就落到上面
        // 的while循环里面去了（while循环里面用的：除法+移位，此处for循环用的：乘法+移位。
        // for循环效率更高），两者就产生了矛盾。所以当精度够高的时候，我们就不移那么多位，保证效率。		 // 例如:
        // 这个里面的i就必须小于: 2的32次方(4294967296)/52429 = 81919。所以就让 i < 2的16次方
        q = (i * 52429) >>> (16+3);
        r = i - ((q << 3) + (q << 1));  // r = i-(q*10)
        buf [--charPos] = digits [r];
        i = q;
        if (i == 0) break;
    }
    if (sign != 0) {
        buf [--charPos] = sign;
    }
}



//这两个数组很巧妙
//十位为0，得到0，十位为1，得到1，无视个位，不用做运算，提高效率
final static char [] DigitTens = {
        '0', '0', '0', '0', '0', '0', '0', '0', '0', '0',
        '1', '1', '1', '1', '1', '1', '1', '1', '1', '1',
        '2', '2', '2', '2', '2', '2', '2', '2', '2', '2',
        '3', '3', '3', '3', '3', '3', '3', '3', '3', '3',
        '4', '4', '4', '4', '4', '4', '4', '4', '4', '4',
        '5', '5', '5', '5', '5', '5', '5', '5', '5', '5',
        '6', '6', '6', '6', '6', '6', '6', '6', '6', '6',
        '7', '7', '7', '7', '7', '7', '7', '7', '7', '7',
        '8', '8', '8', '8', '8', '8', '8', '8', '8', '8',
        '9', '9', '9', '9', '9', '9', '9', '9', '9', '9',
} ;
//个位为0，得到0，个位为1，得到1，无视十位
final static char [] DigitOnes = {
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
} ;
~~~

## toUnsignedString0方法（转二进制、八进制、16进制的核心方法）

~~~java
private static String toUnsignedString0(int val, int shift) {
    // mag为该数二进制串，从第一个为1的位置，到最低位总共的位数
    int mag = Integer.SIZE - Integer.numberOfLeadingZeros(val);
    // chars为存储mag位数所需要的数组长度，shift为进制数以二为底的对数（2-1,8-3,16-4），
    // 也即2进制准换成2进制位，1位用1位存储，转换成8进制，3位用1位存储，16进制4位用1位存储
	// 因为最少都需要1位，所以当前面小于1时，就取1
    // 假设转16进制，mag = 5~8位，都需要两位存储，因为java除法是向下取整的，
    // 所以除了后要加1，但是直接在 mag/shift 后加1,8个2进制位chas = 3出错
    // 所以用 mag 先加 shift-1再除以shift就没问题了 
    int chars = Math.max(((mag + (shift - 1)) / shift), 1);
    char[] buf = new char[chars];

    formatUnsignedInt(val, shift, buf, 0, chars);

    // Use special constructor which takes over "buf".
    return new String(buf, true);
}

	/**
	 * toUnsignedString0核心操作
     */
static int formatUnsignedInt(int val, int shift, char[] buf, int offset, int len) {
    int charPos = len;
    //通过shift计算进制的基
    int radix = 1 << shift;
    //掩码，比进制基少1，即低位全位1。基为8的话，mask = 00...0111(binary)
    int mask = radix - 1;
    do {
        // val & mask 很巧妙，val 与上 mask（假设为111）之后，就得到val的最低3位
        // 再以这个值为索引，取得字符数组里对应的字符
        buf[offset + --charPos] = Integer.digits[val & mask];
        // 上述操作完后，就把val无符号右移3位，继续操作
        val >>>= shift;
    } while (val != 0 && charPos > 0);

    return charPos;
}

final static char[] digits = {
        '0' , '1' , '2' , '3' , '4' , '5' ,
        '6' , '7' , '8' , '9' , 'a' , 'b' ,
        'c' , 'd' , 'e' , 'f' , 'g' , 'h' ,
        'i' , 'j' , 'k' , 'l' , 'm' , 'n' ,
        'o' , 'p' , 'q' , 'r' , 's' , 't' ,
        'u' , 'v' , 'w' , 'x' , 'y' , 'z'
};

~~~

## IntegerCache内部类

~~~java
// 该静态内部类在第一被使用到的时候加载
// 缓存有-128-127(最大值可以设置)，所以用valueOf会首先到这个缓存中寻找
private static class IntegerCache {
    
    //缓存中最小值
    static final int low = -128;
    //缓存中最大值
    static final int high;
    //缓存数组
    static final Integer cache[];

    static {
        // high value may be configured by property
        int h = 127;
        // 从配置中读取high的值
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        // 如果读出的high值不为null
        if (integerCacheHighPropValue != null) {
            try {
                // 把该值解析成为int型
                int i = parseInt(integerCacheHighPropValue);
                // 将较大者复制给i
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                // 数组最大长度不能超过下面，所以对h做出约束
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        // 将最终的h赋值给high
        high = h;
		// 新建缓存数组
        cache = new Integer[(high - low) + 1];
        int j = low;
        // 初始化缓存数组
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);

        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}
~~~

