# [Mr. Phisher](https://tryhackme.com/room/mrphisher)

## I received a suspicious email with a very weird looking attachment. It keeps on asking me to "enable macros". What are those?

I received a suspicious email with a very weird-looking attachment. It keeps on asking me to "enable macros". What are those?

### Uncover the flag in the email attachment!

1. Open `MrPhisher.docm` with LibreOffice Writer.

2. Navigate to `Tools > Macros > Edit Macros`.

3. Expand the project tree to find the `Format` (`MrPhisher.docm > Project > Modules > NewMacros > Format`).

[SCREEN01]

```vb
Rem Attribute VBA_ModuleType=VBAModule
Option VBASupport 1
Sub Format()
Dim a()
Dim b As String
a = Array(102, 109, 99, 100, 127, 100, 53, 62, 105, 57, 61, 106, 62, 62, 55, 110, 113, 114, 118, 39, 36, 118, 47, 35, 32, 125, 34, 46, 46, 124, 43, 124, 25, 71, 26, 71, 21, 88)
For i = 0 To UBound(a)
b = b & Chr(a(i) Xor i)
Next
End Sub
```

4. Decode the array using the Python script

```py
a = [102, 109, 99, 100, 127, 100, 53, 62, 105, 57, 61, 106, 62, 62, 55, 110, 113, 114, 118, 39, 36, 118, 47, 35, 32, 125, 34, 46, 46, 124, 43, 124, 25, 71, 26, 71, 21, 88]
print(''.join(chr(v ^ i) for i, v in enumerate(a)))
```

[SCREEN02]
