# Cross-Site Scripting (XSS)
## 1. Reflected XSS into HTML context with nothing encoded
<img src="./img/reflected/reflected.png" width=400>

### Description
Simple lab with a site that reflects your XSS payload upon submission. Task is to trigger the `alert()` function through a script payload.

### Solution
I used the following script tag and submitted it through the search bar:
> ```js
> <script>alert("hi!!!")</script>
> ```
Immediately, the following pop-up appeared:

<img src="./img/reflected/reflected-popup.png" width=400>

Lab complete.