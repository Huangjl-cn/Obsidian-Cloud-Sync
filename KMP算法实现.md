```java
public class Solution{
	public static int KMP(String text,String pattern){
		int[] next = buildNextArray(pattern);
		int i = 0;
		int j = 0;
		while(j < pattern.length()){
			if(text.length() -i +j <pattern.length()) return -1;
			if(text.chatAt(i) == pattern.charAt(j)){
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
		// prefix[i] 表示长度为 i 的前缀
		String[] prefix = new String[n];
		for(int i = 1;i < n;++i){
			prefix[i] = pattern.substring(0,i);
		}
		
		int[] next = new int[n];
		for(int i = 2;i<n;++i){
			int preLen = i -1;
			while(preLen >0){
				if(pattern.substring(i - preLen,i).equals(prefix[preLen])){
					next[i] = preLen;
					break;
				}
				preLen--;
			}
		}
		return next;
	}
}
```