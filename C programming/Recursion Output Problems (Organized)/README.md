# Recursion Output Problems — সাজানো নোট 📘

মূল ফাইল: `../Recursion output problem.docx` (অক্ষত আছে, কোনো পরিবর্তন করা হয়নি)।
সেই ফাইলের **৮৯টা প্রশ্ন** (কিছু প্রশ্ন ছবিতে ছিল, সেগুলোও টেক্সটে রূপান্তর করা হয়েছে)
টপিক অনুযায়ী ৬টা ফাইলে সাজানো হয়েছে। ডুপ্লিকেট প্রশ্নগুলো এক জায়গায় merge করে নোট দেওয়া আছে।

## 📂 ফাইলগুলো

| ফাইল | বিষয় | প্রশ্ন |
|---|---|---|
| [01 - Tracing & Output Order](01%20-%20Tracing%20&%20Output%20Order.md) | print কল-এর আগে/পরে, LIFO order, call tree ট্রেস | ১২টা |
| [02 - Infinite Recursion & Traps](02%20-%20Infinite%20Recursion%20&%20Traps.md) | base case নেই, `b--` ফাঁদ, negative input, stack overflow | ৮টা |
| [03 - Sum, Series & Factorial](03%20-%20Sum,%20Series%20&%20Factorial.md) | যোগফল, factorial, Fibonacci, power, pseudocode | ২১টা |
| [04 - Digits, Binary & Number Tricks](04%20-%20Digits,%20Binary%20&%20Number%20Tricks.md) | digit sum, binary print, power of 2/3, GCD, prime factor | ১৫টা |
| [05 - Strings & Arrays](05%20-%20Strings%20&%20Arrays.md) | string reverse, vowel count, array search/max | ৮টা |
| [06 - Multiple Calls, Static & Hard](06%20-%20Multiple%20Calls,%20Static%20&%20Hard%20Problems.md) | কল গোনা, mutual recursion, static variable, McCarthy 91 | ১৮টা |

`images/` ফোল্ডারে মূল ডকুমেন্টের সব ব্যাখ্যার ছবি (call tree ইত্যাদি) আছে — নোটের ভেতরে জায়গামতো লিংক করা।

## 📖 কীভাবে পড়বে

প্রতিটি প্রশ্নের ফরম্যাট সহজ: **প্রশ্ন → কোড → অপশন → ✅ উত্তর → ব্যাখ্যা** — সব সরাসরি দেখা যায়। VS Code-এ সুন্দর করে দেখতে Preview mode-এ খুলো (`Ctrl+Shift+V`)।

## ⚡ Quick Revision — পরীক্ষার আগে এটুকু মনে রাখো

1. **print রিকার্সিভ কলের আগে** → নামার পথে প্রিন্ট (10 9 8...)
   **print রিকার্সিভ কলের পরে** → ফেরার পথে প্রিন্ট, উল্টা ক্রম (LIFO: 0 1 2 3)
2. **`f(b--)`** → post-decrement, পুরনো মানই পাস হয় → **infinite loop!** (`f(--b)` বা `f(b-1)` হলে ঠিক)
3. **base case নেই / পৌঁছানো যায় না** (negative input, odd→even mismatch) → **stack overflow, runtime error**
4. **`n%10 + f(n/10)`** → digit sum | **`n%2` প্রিন্ট + `f(n/2)`** → binary (আগে প্রিন্ট = উল্টা, পরে প্রিন্ট = সোজা)
5. **`f(n/2)*10 + n%2`** → বাইনারি রূপ দশমিকের চেহারায় (magicfun)
6. **`n + f(n-1)`** → যোগফল n(n+1)/2 | **`n * f(n-1)`** → factorial
7. **static variable** → সব কলে একটাই কপি, reset হয় না — ফেরার পথেও সর্বশেষ মান দেখায়
8. **কল গোনার প্রশ্নে** base case-এর কলটাও গুনতে হয় (f(5) হলে f(0) পর্যন্ত = 6 কল)
9. **string reverse কৌশল** = আগে কল, পরে putchar — অক্ষর stack-এ জমে উল্টা বের হয়
10. **fun(fun(n+11))**, n≤101 → সবসময় 91 (McCarthy 91 function)

## ⚠️ মূল নোটে পাওয়া কিছু সন্দেহজনক জায়গা

- **০৬-এর প্রশ্ন ১৮** (mystery(3)): ট্রেস করলে 12 আসে, মূল নোটে উত্তর 14 — নিজে যাচাই করো।
- **০৩-এর প্রশ্ন ১৫** (my_function(6,10)): নিয়মমতো ট্রেস করলে 40 আসে, কিন্তু অপশনে 40 নেই — মূল নোটের উত্তর 35।
