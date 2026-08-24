## Cipher কী?

**Cipher** হলো একটি পদ্ধতি যার মাধ্যমে একটি পাঠযোগ্য message (যাকে বলে **plaintext**) কে এলোমেলো, অপাঠযোগ্য রূপে (যাকে বলে **ciphertext**) রূপান্তর করা হয়, যাতে শুধুমাত্র যে ব্যক্তি গোপন নিয়মটা (**key**) জানে, সে-ই message টা পড়তে পারে। এই পুরো বিষয়টাকে বলা হয় **Cryptography**।

```
Plaintext:  HELLO   →  (cipher + key প্রয়োগ)  →  Ciphertext: KHOOR
```

"Cipher programming" মানে হলো এমন একটি program লেখা যা এই encryption এবং decryption-এর কাজ করে।

## পরীক্ষায় সবচেয়ে বেশি আসে: Caesar Cipher

এটাই সবচেয়ে বেশি দেখা যায়। নিয়মটা হলো: **প্রতিটি letter-কে একটি নির্দিষ্ট সংখ্যক ঘর সামনে সরিয়ে দেওয়া** (এই সংখ্যাটাই key)।

Key = 3 হলে:

- A → D, B → E, C → F ... X → A, Y → B, Z → C (শেষে গিয়ে আবার শুরু থেকে ঘুরে আসে — একে বলে wrap around)
- তাই `HELLO` → `KHOOR`

Decrypt করতে হলে ৩ ঘর পেছনে সরাতে হবে।

## বাংলাদেশের সরকারি IT পরীক্ষায় কীভাবে প্রশ্ন আসে

সাধারণ প্রশ্নের ধরন:

১. **সরাসরি encryption (সবচেয়ে common):** "Caesar cipher-এ shift 3 ব্যবহার করে 'EXAM' এর ciphertext কী হবে?" → উত্তর: `HADP`

২. **Decryption:** "Shift 3 দিয়ে তৈরি ciphertext থেকে plaintext বের করুন" — প্রতিটি letter পেছনে সরিয়ে সমাধান করতে হয়।

৩. **Theory MCQ:**

- "Encrypted text-কে কী বলে?" → Ciphertext
- "কোনটি symmetric key algorithm?" → AES / DES (encrypt ও decrypt-এ একই key)
- "কোনটি asymmetric algorithm?" → RSA (public key + private key)
- "Caesar cipher কোন ধরনের cipher?" → Substitution cipher

৪. **ছোট code-এর output প্রশ্ন:** একটি C program দেখিয়ে জিজ্ঞেস করা হয় output কী হবে — যেখানে character-এর সাথে `ch + 3` করা হয়েছে।

## মুখস্থ রাখার মতো গুরুত্বপূর্ণ Theory Term

|Term|অর্থ|
|---|---|
|Plaintext|মূল পাঠযোগ্য message|
|Ciphertext|Encrypted message|
|Key|Encrypt/decrypt করার গোপন মান|
|Substitution cipher|প্রতিটি letter-কে অন্য letter দিয়ে বদলে দেয় (Caesar, Vigenère)|
|Transposition cipher|Letter-এর অবস্থান এলোমেলো করে, letter বদলায় না|
|Symmetric encryption|দুই দিকেই একই key — DES, AES|
|Asymmetric encryption|Public key দিয়ে encrypt, private key দিয়ে decrypt — RSA|

## Code-এ Caesar Cipher

**Python (শেখার জন্য সবচেয়ে সহজ):**

```python
def encrypt(text, key):
    result = ""
    for ch in text:
        if ch.isalpha():
            base = ord('A') if ch.isupper() else ord('a')
            result += chr((ord(ch) - base + key) % 26 + base)
        else:
            result += ch
    return result

def decrypt(text, key):
    return encrypt(text, -key)

print(encrypt("HELLO", 3))   # KHOOR
print(decrypt("KHOOR", 3))   # HELLO
```

**C (বাংলাদেশের পরীক্ষায় সাধারণত C ব্যবহার হয়):**

```c
#include <stdio.h>
#include <ctype.h>

int main() {
    char text[100];
    int key = 3;

    printf("Enter text: ");
    scanf("%s", text);

    for (int i = 0; text[i] != '\0'; i++) {
        if (isupper(text[i]))
            text[i] = (text[i] - 'A' + key) % 26 + 'A';
        else if (islower(text[i]))
            text[i] = (text[i] - 'a' + key) % 26 + 'a';
    }
    printf("Encrypted: %s\n", text);
    return 0;
}
```

দুটো code-এরই মূল কৌশল এই line-এ: `(letter - 'A' + key) % 26 + 'A'`। এখানে `% 26` এর কাজ হলো Z-এর পরে আবার A-তে ফিরে আসা। এই একটি line ভালোভাবে বুঝলে আপনি Caesar cipher-এর যেকোনো প্রশ্নের উত্তর দিতে পারবেন।

## পরীক্ষার মতো করে Practice করুন

হাতে-কলমে এগুলো চেষ্টা করুন (উত্তর নিচে):

১. `DHAKA` কে shift 2 দিয়ে encrypt করুন ২. `ORYH` কে shift 3 দিয়ে decrypt করুন ৩. Caesar cipher কোন ধরনের cipher — substitution নাকি transposition?

**উত্তর:** ১. `FJCMC` ২. `LOVE` ৩. Substitution।

আরেক ধাপ গভীরে যেতে চাইলে (মাঝে মাঝে written পরীক্ষায় আসে), পরের topic হলো **Vigenère cipher** (একটি সংখ্যার বদলে একটি keyword দিয়ে Caesar cipher) এবং **RSA**-এর মূল ধারণা (শুধু theory — code লিখতে বলা হবে না)। চাইলে এই দুটোর যেকোনোটা বুঝিয়ে দিতে পারি, অথবা BD পরীক্ষার format-এ আরও বেশি practice প্রশ্ন দিতে পারি — শুধু বলবেন।