```java
public class Solution{
    public static int KMP(String text,String pattern){
        int[] next = buildNextArray(pattern);
        int i = 0;
        int j = 0;
        while(j < pattern.length()){
            if(text.length() -i +j <pattern.length()) return -1;
            if(text.charAt(i) == pattern.charAt(j)){
                i++;
                j++;
            }else if(j == 0){
                i++;
            }else{
                j = next[j];
            }
        }
        return i - j;
    }

    public static int[] buildNextArray(String pattern){
        int n = pattern.length();
        // next[i] 表示 pattern[0..i-1] 的最长公共前后缀长度
        // next[0]、next[1] 按定义为 0，默认初始化即可
        int[] next = new int[n];
        for(int i = 2;i < n;++i){
            // 从上一个位置的最长 border 开始尝试延长
            int k = next[i - 1];
            // border 的 border 还是 border：失配就退到 next[k] 再试
            while(k > 0 && pattern.charAt(i - 1) != pattern.charAt(k)){
                k = next[k];
            }
            if(pattern.charAt(i - 1) == pattern.charAt(k)){
                k++;
            }
            next[i] = k;
        }
        return next;
    }
}
```