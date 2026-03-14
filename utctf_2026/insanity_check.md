# Writeup: Insanity Check (UTCTF 2026)

## Challenge Overview
The "Insanity Check" challenge appeared at first to be a simple "Sanity Check" style task, but it hid its flag behind several layers of obfuscation and "hidden" content within the CTFd platform.

## My Solve Process

### 1. Initial Reconnaissance
I started by exploring the files provided in the challenge directory.
- `challenges.json`: A simple redirect to the login page.
- `homepage_full.txt`: The full source of the CTF homepage.

Looking closely at `homepage_full.txt`, I noticed some suspicious CSS and JavaScript:
```html
<style id="theme-color">
  .pt-5:has(.challenge-button[value='244']) { display: none; }
</style>
<script>
const callback = (list, observer) => {
  document.querySelector(".challenge-button[value='244']")?.closest(".pt-5")?.remove();
  observer.disconnect();
};
document.addEventListener("DOMContentLoaded", e => {
  const observer = new MutationObserver(callback);
  observer.observe(document.querySelector('[x-data="ChallengeBoard"]'), { childList: true, subtree: true });
});
</script>
```
It seemed the organizers were actively trying to hide challenge ID `244` from the UI.

### 2. Discovering the Clues
I then checked the other files in the directory, specifically `path1.txt` and `path2.txt`. These files appeared to be 404 pages from the CTF platform, but they held the key to the solution.

In `path1.txt`, I found a list of numbers hidden in an HTML comment:
```html
<h1>File not found</h1> <!-- 2, 7, 9, 7, 8, 13, 17, 39, 85, 4, 57, 4, 93, 30, 104, 27, 44, 23, 89, 8, 30, 68, 107, 112, 54, 0, 30, 11, 2, 92, 66, 23, 31 -->
```

In `path2.txt`, there was another list of numbers in a similar comment:
```html
<h1>File not found</h1> <!-- 119, 115, 111, 107, 105, 106, 106, 110, 114, 105, 102, 106, 50, 106, 55, 122, 115, 101, 54, 106, 113, 48, 52, 57, 105, 112, 108, 100, 111, 53, 49, 114, 98 -->
```

### 3. Exploitation
The presence of two lists of similar length immediately suggested an XOR operation. I wrote a quick Python script to XOR the two sequences and see if it revealed the flag.

```python
path1 = [2, 7, 9, 7, 8, 13, 17, 39, 85, 4, 57, 4, 93, 30, 104, 27, 44, 23, 89, 8, 30, 68, 107, 112, 54, 0, 30, 11, 2, 92, 66, 23, 31]
path2 = [119, 115, 111, 107, 105, 106, 106, 110, 114, 105, 102, 106, 50, 106, 55, 122, 115, 101, 54, 106, 113, 48, 52, 57, 105, 112, 108, 100, 111, 53, 49, 114, 98]

flag = "".join(chr(p1 ^ p2) for p1, p2 in zip(path1, path2))
print(flag)
```

Running this script gave me the flag.

### 4. Final Result
The XOR operation revealed the flag:
`utflag{I'm_not_a_robot_I_promise}`

## Conclusion
The challenge was a clever scavenger hunt that required looking past the surface of the "broken" parts of the site and realizing that the 404 pages were actually providing the XOR keys for the final flag.
