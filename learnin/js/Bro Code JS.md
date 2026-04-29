console.log(`adf`)
window.alert(`what is your name?`)
//this is comment
/*whatt is your name*/
``` js
```


document, getelementbyid("my h1").textcontent -=`hello`


we use the consoel to show the variable andn or use the window alert to do the same thing personally i recomned to use the windows alert because i think that look good

second thing you should recommend is that what sellecting the div we shoudl first give the id to that div like:

```javascript
document.getElementById(""myH1).textContent = 'Helllo';
```

```html
<h1 id="myH1"> </h1>
```

so first we access the document of html, go to his id which matches the `myH1` then chane the textContent to change to `Hello`

code template in the js is this 
```js
console.log("Is your name ${Hamza}");
```

to check the type of the variable you can use the typeof funciton

like this 
`console.log(typeof firstName);`

**athematic operators**
operands & operators

*now we should also know about the augmented assignment operators*
like `students += 1`

**type conversion**
when we take the value from the user in the number by not setting the type what type it gonna be then we by default and that's what typescript use for

so if we take the value from the user we should know that what is the `typeof` of the variable so we can know it's use case

we can set the input from the user to from the string or number to Boolean so we can know is the user write something there or not

**constant**
if you try to assignment again to the constant, it through the error, the code will break so do work on this to handle it gracfully

it is a good practice to make the buttons and function in the const because nobody can inject function in it, second is that we need to use it in the multiple places 

```js
const decreaseBtn = document.getElementById("decreasbtn");
// it will still work even if we don't keep in the const but the SECURITY HMMM!
document.getElementById("decreasbtn");
```
**MATHS**
built-in library
like pi, e and others
```
Math.Pi
```

**if & else**
*innerHTML* is unsafe because we can do the injection in it
*textContent* is secure we can't paste the tag in it

checkbox has the property which is checked and can be access like this 
```js
myCheckbox.checkbox
```

**Ternary operator**
```
condition ? true : false;
```

**String Methods (it help a lot)**
 ```js
 userName.charAt
 userName.indexOf(0) ==> first occurrence of the letter
 userName.lastIndexOf("k") ==> last occurence of the letter
 userName.slice
 userName.trim
 userName.uppercase
 userName.length
 userName.toUppercase
 userName.toLowerCase
 userName.repeat(2)
 userName.startsWith("a") ==> true, false
 userName.endsWith(0)
 userName.includes(" ") 
 phoneNumber.replaceAll("-". "")
 phoneNumber
 phoneNumber.padStarts(15, "0")
 phoneNumber.padEnds(15, "0")
 ```

*String Slicing*
```js
string.slice(start, end)
```

 