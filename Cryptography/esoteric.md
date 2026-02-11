### Challenge description :

<img width="662" height="773" alt="image" src="https://github.com/user-attachments/assets/36646f31-9e08-4cf5-9900-077748f733be" />

### Difficulty : Eassyyyy

<img width="554" height="188" alt="image" src="https://github.com/user-attachments/assets/0218b1a0-fc6e-4b49-b186-f8cdb2dda9a0" />


Password : esoteric (the challenge title - just a random guess)

We started by analyzing the provided image `images.jpeg`. Checking the metadata with `exiftool` revealed a suspicious string in the **Caption-Abstract** field.

<img width="835" height="520" alt="image" src="https://github.com/user-attachments/assets/496ffb46-e346-4fe5-8096-cd00110a0a3e" />

**Found:** `cyb3rs3c — ref id: 23xA_63.72.40.63.6b.5f.6d.33x095`

The string `63.72.40.63.6b.5f.6d.33` is Hexadecimal. Decoding it reveals a password:

- `0x63 72 40 63 6b 5f 6d 33` -> **`cr@ck_m3`**

Using the password found in the metadata, we checked for hidden files inside the image using `steghide`.

<img width="851" height="121" alt="image" src="https://github.com/user-attachments/assets/b33a5250-2ed1-4f61-aa36-6132869962ba" />


<img width="642" height="699" alt="image" src="https://github.com/user-attachments/assets/b2d30230-8e20-49c8-9043-a277af41cc16" />


Putting it all together, the decoded message spells `HELLO_SOLDIER`.

**Flag:** `SECE{HELLO_SOLDIER}`
