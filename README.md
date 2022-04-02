# MWiki

#### Introduce
The memory flowing through the fingertip is both a moment and an eternity.

#### Instruction

Welcome to my study wiki.

public static void main(String[] args) {
        String str1 = "11111111";
        String str2 = "00000001";
        int i = Integer.parseInt(str1, 2);
        int j = Integer.parseInt(str2, 2);
        int result = i & j;
        String s = Integer.toBinaryString(result);
        System.out.println(s);
    }
    
