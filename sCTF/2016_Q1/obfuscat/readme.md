# Obfus(cat)ion

###### Note: This write-up is written that you can solve this puzzle by just using a browser and your brain ;-)

When opening the page we were presented with the GIF of a cat.
After a quick look at the HTML source-code we were able to identify the relevant javascript that was used to check the input.

We also saw that the embedded GIF was hard-coded as a base64 string, but this is not important at all.

Let's go and check the Obfuscated JS-code (The image was borrowed from "https://wordswithcomputers.files.wordpress.com" as I just had a modified version and the web-site was not available anymore):

![Obfus-cat-ion JS code](https://wordswithcomputers.files.wordpress.com/2016/04/sctf_obfuscat1.png)

Inside the HTML was a placeholder for the text `sctf{...}` which suggested that the flag starts with `sctf{` (as usual).

## Pseudo-Randomness

When looking at the obfuscated JS it becomes obvious that there is some pseudo-randomness involved via the `MersenneTwister` objects.
The PRNGs are always seeded with the same values (and subsequent calls to .random() deliver the same values if the call structure is reproduced)
The following lines handle the "pseudo-randomness" and the causality between them:

```
h = new MersenneTwister(parseInt(btoa(answer.substring(0, 4)), 32));				// The new MersenneTwister is initialized with the int representation of "sctf"
e = h[_[$[""+ +[]]]]()*(""+{})[_[0x4728122]](0xc);
// equivalent to: h.random()*"[object Object]".charCodeAt(12)						// e is a constant value (e = 22.669553810264915)
for(var _1=0; _1<h.mti; _1++) { e ^= h.mt[_1]; }									// e = 54.33253672812134
l = new MersenneTwister(e), v = true;												// l = new MersenneTwister(54.33253672812134), v = true
```

As you can see, several weird objects are used (_, $). Those objects are declared on the very top of the JS-Section and are primarily for confusion purposes. They encode functions like "charCodeAt".

The first two PRNGs are really deterministic (`h` and `l` ) and do not depend on the actual text inside the flag, but of the four letters `sctf`.

Before further discussing the code, I want to introduce another function which is used in the next code snippet.
We will call this function "toHexString()":

```
function toHexString(s) {
	var charArray = s.split('');
	var result = "0x";
	
	for(i = 0; i < charArray.length; i++) {
		result += s.charCodeAt(i).toString(16)
	}
	return result;
}
```

The "toHexString" function takes a string as an input parameter and converts it to its HEX representation. (e.g.: "AAAA" becomes "0x41414141", "AAABB" becomes "0x4141414242").
This function is used in the next code segment to derive the starting value of the next PRNG.

The following code is responsible for ensuring that the second word of the flag (inside the "{}" is correct):

```
l.random(); l.random(); l.random();													// three "NOPs" (increase next random value)
o = answer.split("_");																// input is split by "_" into an array
i = l.mt[~~(h.random()*35725343)%255];												// i is deterministic: 941574242
s = ["0x" + i.toString(0x10), "0x" + e.toString(0o20).split("-")[1]];				// s is deterministic: [0x381f4862,0x3a9b9622]
e = -(this[_[$[42]]](_[$[31]](o[1])) ^ s[0]);if (-e != $[21]) return false;
// equivalent to: e =- (this.eval(_[35725343](o[1])) ^ s[0]);if (-e != $[21]) return false;
// equivalent to: e = e - (this.eval(toHexString(o[1])) ^ 0x381f4862); if(-e != $[21]) return false;
// equivalent to: e = e - (this.eval(toHexString(o[1])) ^ 0x381f4862); if(-e != 941564184) return false;
```

As you can see, the condition that needs to be fulfilled is: ` - (this.eval(<HEX-String of second Flag-word>) XOR 0x381f4862) == -941564184`

This formula can be re-written to be: `941564184 ^ 0x381f4862 == this.eval(<HEX-String of second Flag-word>)``
After rewriting and calculating we get: `this.eval(<HEX-String of second Flag-word>) == 27002`

The `this.eval` function converts a HEX-String to an int. Therefore we are able to reconstruct the HEX values from the int value with `parseInt("27002").toString(16)`.
Now we know that the HEX value is `697a`, which we know is (after a look inside the ASCII table): **iz**.

###### Here we go: The second word inside our flag is "iz"

Now let's check on the third word inside our flag which is covered by the next line of code:
```
e ^= (this[_[$[42]]](_[$[31]](o[2])) ^ s[1]); if (-e != $[22]) return false; e -= 0x352c4a9b;
// equivalent to e = e ^ (this.eval(_[35725343](o[2])) ^ s[1]); if (-e != $[22]) return false;
// equivalent to e = e ^ (this.eval(toHexString(o[2])) ^ 0x3a9b9622); if (-e != 48879197) return false;
```

This code snippet checks the correct spelling of the third word, which needs the following condition to be satisfied:
`e ^ (this.eval(<HEX-String of third Flag-word>) ^ 0x3a9b9622) == -48879197`

From our previous line of code we know that e has the value `-941564184` when the Instruction pointer reaches this line.

Therefore we can simplify our formula to the following:
`this.eval(<HEX-String of third Flag-word>) == -48879197 ^ (-941564184) ^ 0x3a9b9622`

When calculating this formula the result is an int with value `7168361`. After converting it to HEX (`parseInt("7168361").toString(16)`) and looking the result up in the ASCII Table we can identify the second word as **mai**.


##### Summary

Up to this point we know that the flag must be of the form `sctf{?????_iz_mai_?????}` where the question marks mark unknown letters (including unknown length of the word)


#### Finish the Pseudo-randomness
```
e -= 0x352c4a9b;									// after this command e is: -940974328
t = new MersenneTwister(Math.sqrt(-e));				// so the new PRNG is initialized with a known seed. :-)
```

Up to now we were able to reconstruct the whole "randomness" in this JS-code.

Let the code continue:
```
h.random();											// another "NOP" (skip one random value)
a = l.random();										// deterministic random value is assigned to "a": 0.03319016075693071
t.random();											// another "NOP"
y = [ 0xb3f970, 0x4b9257a, 0x46e990e ].map(function(i) { return $[_[$[40]]](i)+ +1+ -1- +1; });
// equivalent to: y = [ 0xb3f970, 0x4b9257a, 0x46e990e ].map(function(i) { return $.indexOf(i) -1;})
// result: 1,2,4
o[0] = o[0].substring(5);							// the first word is shortened (the "sctf{" is removed)
o[3] = o[3].substring(0, o[3].length - 1);			// the closing curly brace is removed from the last word
u = ~~~~~~~~~~~~~~~~(a * i);						// irrelevant line of code
// if someone looks at the source code u is never used again.
if (o[0].length > 5) return false;					// The first word inside the curly braces must have less than 6 letters, otherwise the JS-code returns false.
```


##### Ok, let's summarize

1. We know that the flag starts with `sctf{` and ends with `}`
2. We know the flag contains at least three `_`
3. We know that the second word is `iz`
4. We know that the third word is `mai`
5. We know that the first word has <= 5 letters


Therefore we know already a lot of the flag. :-)

... and 5 letters could be brute-forced (could it?).

## The hard part

Until now the reversing was pretty straightforward. But now comes the hard part.

I will first discuss the structure of this section before jumping in:
1. Give an overview of the snippet
2. Discuss different parts of the code
3. Put all the pieces together

The second sub-section will contain a lot of jumping between code lines, therefore concentration is needed. ;-)

### Snippet overview

The remaining part of the code is listed below:
```
//_[$[23]] prints the string in the first parameter as often as the int in the second parameter says
a = parseInt(_[$[23]]("1", Math.max(o[0].length, o[3].length)), 3) ^ eval(_[$[31]](o[0]));
r = (h.random() * l.random() * t.random()) / (h.random() * l.random() * t.random());
e ^= ~r;
r = (h.random() / l.random() / t.random()) / (h.random() * l.random() * t.random());
e ^= ~~r;
a += _[$[31]](o[3].substring(o[3].length - 2)).split("x")[1];
if (parseInt(a.split("84")[1], $.length/2) != 0x4439feb) return false;
d = parseInt(a, 16) == (Math.pow(2, 16)+ -5+ "") + o[3].charCodeAt(o[3].length - 3).toString(16) + "53846" + (new Date().getFullYear()- +1+ "");
i = 0xff;
n = (p = (f = _[$[23]](o[3].charAt(o[3].length - 4), 3)) == o[3].substring(1, 4));
g = 111;
t = _[$[23]](o[3].charAt(3), 3) == o[3].substring(5, 8) && (o[3].charCodeAt(1)-2) * o[0].charCodeAt(0) == 0x32ab;
h = ((g ^ e ^ 96) & i).toString(16);
i = o[3].split(f).join("");			
s = i.substring(0, 2) == h;
return (p & t & s & d) === 1 || (p & t & s & d) === true;
```

### Part discussion

We will now start to discuss the different parts of the code.
To give a good overview we will work from bottom to top.

###### Note: Simplifications have been made to increase readability

```
r = (h.random() * l.random() * t.random()) / (h.random() * l.random() * t.random());
e ^= ~r;
r = (h.random() / l.random() / t.random()) / (h.random() * l.random() * t.random());
e ^= ~~r;																// e is a constant value of 940974335

[...] supressed for brevity

f = _[$[23]](o[3].charAt(o[3].length - 4), 3)

[...] supressed for brevity

i = 0xff;
g = 111;
h = ((g ^ e ^ 96) & i).toString(16);
// equal to: ((111 ^ 940974335 ^ 96) & 0xff).toString(16)
// equal to: h = "f0"
i = o[3].split(f).join("");			
s = i.substring(0, 2) == h;
return (p & t & s & d) === 1 || (p & t & s & d) === true;
```

In the last line we see that `s` must be true in order to get a valid result.
Therefore we will now start inspecting `s`.
Before we can calculate `s` we need to have the following values: `i, h, g, i, e`

As you see on the very top of the code snippet, the value of e is calculated using .random() functions. And we already know the seed values and how often `random()` was called. Therefore we are able to reconstruct `e`.
The calculated value for `e` was alredy put into the comments of the previous code. Please see there for reference.

Also the calculation of `h` has been covered in the code. Therefore we will discuss on `i` and `f`.

Let's first focus on `f`:

```
f = _[$[23]](o[3].charAt(o[3].length - 4), 3)
```

The function `_[$[23]]` is encoded and is a helper function which corresponds to "RepeatCharacterXTimes". The code is outlined here:
```
function RepeatCharacterXTimes(character, howOften) {
	var result = "";
	for(var k = 0; k < howOften; k++) {
		result += character;
	}
	return result;
}
```
After knowing this, the code for `f` can be translated to:

```
f = RepeatCharacterXTimes(o[3].charAt(o[3].length - 4), 3)
```

This means, that the character that is present at the 4th last position of the fourth word inside the flag is repeated 3 times.
e.g. if the flag would be `sctf{asdf_iz_mai_hoooooooooooooouome}` then `f` would be equal to `"uuu"`.

So up to now we know that the value of `h` is `"f0"`, and how `f` is calculated.

```
i = o[3].split(f).join("");			
s = i.substring(0, 2) == h;
```

From the first line in this code snippet it is obvious that the value stored in `f` is removed from the fourth word of the flag.
And the first two letters of this new string must be equal to `"f0"` in order to fulfill this requirement.

##### Summary:
* The fourth word of the flag must start with "f"
* The 5th letter of the fourth word of the flag must be "0"


### Part discussion continued

So now we have some information on the structure of the fourth word inside the flag.
Now we will look at additional constraints for the first and fourth word inside the flag:


```
//_[$[23]] prints the string in the first parameter as often as the int in the second parameter says
a = parseInt(_[$[23]]("1", Math.max(o[0].length, o[3].length)), 3) ^ eval(_[$[31]](o[0]));

a += _[$[31]](o[3].substring(o[3].length - 2)).split("x")[1];
if (parseInt(a.split("84")[1], $.length/2) != 0x4439feb) return false;
d = parseInt(a, 16) == (Math.pow(2, 16)+ -5+ "") + o[3].charCodeAt(o[3].length - 3).toString(16) + "53846" + (new Date().getFullYear()- +1+ "");
```

If you look at the `if` statement in above code block, it becomes clear that the value of `a.split("84")[1]` can be calculated, because we also know that the value of `$.length/2` equals to 25.

This can be done by converting the value `0x4439feb` to base 25 (e.g. via: https://www.tools4noobs.com/online_tools/base_convert/)
The result is `783f3f`.

Therefore we know that the string of `a` must end with "84783f3f" before the if statement succeeds.
One line above the `if` statement a string of two characters (the last two characters of the fourth word) is converted to HEX and appended to `a`. In order to be ok for the check in the `if` statement the value that is appended must be "3f3f" which corresponds to `??` in ASCII.

###### Another thing we noticed: The fourth word must end with `??`

To further reconstruct `a` we will focus on the line beneath the `if` statement:

```
d = parseInt(a, 16) == (Math.pow(2, 16)+ -5+ "") + o[3].charCodeAt(o[3].length - 3).toString(16) + "53846" + (new Date().getFullYear()- +1+ "");
```

This line asserts that for d to become `true`, the value of `a` (converted from HEX) must be equal to the string "65531??538462015" where the `?` denote unknown values.

From our previous knowledge (that `a` must end with "84783f3f") we can bruteforce the two missing digits with the following JavaScript code (paste it in your Browser Debugger):
```
for(var i = 0; i < 10; i ++){
	for(var j = 0; j < 10; j++) {
		if(parseInt("65531" + i + j + "538462015").toString(16).endsWith("84783f3f")) {
			console.log("found the missing numbers: " + i + j);
		}
	}
}
```

Therefore we know that the third last letter of the fourth word must be `d` (64 in HEX).

When looking at the next statement (that one we didn't covered up to now) we can get additional info on the first and fourth words:
```
t = _[$[23]](o[3].charAt(3), 3) == o[3].substring(5, 8) && (o[3].charCodeAt(1)-2) * o[0].charCodeAt(0) == 0x32ab;

// simplifies to:

t = RepeatCharacterXTimes(o[3].charAt(3), 3) == o[3].substring(5, 8) && (o[3].charCodeAt(1)-2) * o[0].charCodeAt(0) == 0x32ab;

```

Let's discuss the first part of this statement: `RepeatCharacterXTimes(o[3].charAt(3), 3) == o[3].substring(5, 8)`

We know from our earlier investigation that the fourth word starts with `f___0` where `_` denotes an unknown letter.
Therefore this part of the condition asserts that the characters 5,6,7 of the fourth word must be equal to the characters 1,2,3.
This is a good hint for us.

Therefore the fourth word starts with `f___0___` where `_` denotes an unknown letter and ends with `d??`.

If we now take a look at the second part of the condition (`(o[3].charCodeAt(1)-2) * o[0].charCodeAt(0) == 0x32ab`) we notice that we can easily bruteforce this.
Because we know that o[3].charCodeAt(1) is one of our unknown letters inside the fourth word.

The following JS-Snippet can be used to recover possible combinations:

```
for(var i = 35; i < 128; i ++){
	for(var j = 33; j < 126; j++) {
		if((i-2) * j == 0x32ab) {
			console.log("o[3].charCodeAt(1): " + String.fromCharCode(i) + "; o[0].charCodeAt(0): " + String.fromCharCode(j));
		}
	}
}
```

The output we get is the following:
```
o[3].charCodeAt(1): o; o[0].charCodeAt(0): w
o[3].charCodeAt(1): y; o[0].charCodeAt(0): m
```

#### Summary:
* The fourth word of the flag must start with "f"
* The 5th letter of the fourth word of the flag must be "0"
* The last three letters of the fourth word are "d??"
* The fourth word starts with the string `f___0___` where `_` denotes an unknown letter
* The first word can just start with "m" or "w"
* The first word must have fewer than 6 letters
* If the first word starts with "w" our missing letter in the fourth word is "o"
* If the first word starts with "m" our missing letter in the fourth word is "y"


### Part analysis continued

Let's finally focus on a really important line of code which will help us uncover the secrets of the flag:
```
a = parseInt(_[$[23]]("1", Math.max(o[0].length, o[3].length)), 3) ^ eval(_[$[31]](o[0]));
// equal to:
a = parseInt(RepeatCharacterXTimes("1", Math.max(o[0].length, o[3].length)), 3) ^ eval(toHexString(o[0]));
```

As we know, that the length of the first word must be <= 5 and that the fourth word has definetly more than 5 characters we can further simplify the line to:
```
a = parseInt(RepeatCharacterXTimes("1", o[3].length), 3) ^ eval(toHexString(o[0]));
```

This line of code repeats the string "1" for the amount of letters inside the fourth word. The resulting string (e.g. "1111") is converted to an int from base 3 and afterwards XORed with the HEX representation of the first word.

The length of the first word must be less or equal to 5 characters, therefore the following snippet can help us to bruteforce the result.
At this point of time we also know that the value of `a` must be `1748118478`.

```
function HexStringToString(s) {
	var result = "";
	for (var j = 0; j < s.length; j+=2) {
		result += String.fromCharCode(parseInt(s[j] + s[j+1],16));
	}
	return result;
}

for(var i = 8; i < 15; i++){
	var myVal = parseInt(RepeatCharacterXTimes("1",i),3) ^ 1748118478
	console.log("i = " + i + ", value: " + HexStringToString(myVal.toString(16)));
}
```

The result we get is the following:
```
i = 8, value: h2'
i = 9, value: h2¿
i = 10, value: h2X
i = 11, value: h3r3
i = 12, value: h6&6
i = 13, value: h>'
i = 14, value: hVr
```

With our previous knowledge, that the first letter can just be "w" or "m" the length of 11 would make sense `wh3r3`.

This means that we know the length of the fourth word and the first word of the flag. With the first letter being a "w" we also know that the missing letters in the fourth word correspond to `o`.

###### Notice: Now the flag starts with `sctf{wh3r3_iz_mai_fooo0ooo` and ends with `d??}`.

#### Summary:

We know that the fourth word conatins 11 letters, and `fooo0ooo` are already 8 and `d??` are 3 letters we can construct the fourth word as `fooo0oood??`.

And the flag is: `sctf{wh3r3_iz_mai_fooo0oood??}`

### Conclusion

The Obfuscation challenge was a lot of fun, and I personally learned a lot in terms of source code obfuscation for JS.

But **Obfuscation != Encryption **.

And this were 180 points for us :P

Cheers,
sternze
(member of scan.net)