```markdown
# CyberHeroes — Write-Up

**Target:** 10.82.159.203  
**Date:** 13-12-2025  
**Author:** Mohammad Meraj

---

## Reconnaissance

Initial port and service enumeration was performed using Nmap:

```bash
nmap -sS -sV -oN nmap/initial 10.82.159.203 -T4
```
![alt text](<Images/Screenshot 2025-12-13 073823.png>)

### Results

| Port | State | Service | Version                         |
| :--- | :---: | :-----: | :------------------------------ |
| 22   | open  |   ssh   | OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 |
| 80   | open  |   http  | Apache httpd 2.4.48 (Ubuntu)    |

**Service Info:**

* OS: Linux

---

## Web Enumeration

After accessing the web service on port **80**, the page functionality appeared to be a simple authentication form.

Initial login attempts through the UI did not succeed.

---

## Source Code Inspection

Since the intended path failed, the page source was inspected. The following JavaScript authentication logic was discovered:

```javascript
function authenticate() {

  a = document.getElementById('uname')
  b = document.getElementById('pass')

  const RevereString = str => [...str].reverse().join('');

  if (a.value=="h3ck3rBoi" & b.value==RevereString("54321@terceSrepuS")) { 

    var xhttp = new XMLHttpRequest();

    xhttp.onreadystatechange = function() {
      if (this.readyState == 4 && this.status == 200) {
        document.getElementById("flag").innerHTML = this.responseText ;
        document.getElementById("todel").innerHTML = "";
        document.getElementById("rm").remove() ;
      }
    };

    xhttp.open(
      "GET",
      "RandomLo0o0o0o0o0o0o0o0o0o0gpath12345_Flag_"
      + a.value + "_" + b.value + ".txt",
      true
    );
    xhttp.send();
  }
  else {
    alert("Incorrect Password, try again.. you got this hacker !");
  }
}
```
![alt text](<Images/Screenshot 2025-12-13 081033.png>)
---

## Credential Analysis

* **Username:** `h3ck3rBoi`
* **Password logic:** reverse of `"54321@terceSrepuS"`

Reversing the string results in:

```
SuperSecret@12345
```

---

## Exploitation

Using the derived credentials:

* The authentication condition was satisfied.
* A hidden request was made to a predictable flag file.
* The flag was successfully displayed on the page.

![alt text](<Images/Screenshot 2025-12-13 080932.png>)
---

## Conclusion

This challenge relied on:

* Basic reconnaissance
* Logical troubleshooting when the initial approach failed
* Client-side code inspection
* Understanding and reversing simple string operations

While technically simple, the correct methodology was followed:
**attempt → observe → inspect → understand → exploit**.
