# [Epoch](https://tryhackme.com/room/epoch)

## Be honest, you have always wanted an online tool that could help you convert UNIX dates and timestamps!

# Epoch

Be honest, you have always wanted an online tool that could help you convert UNIX dates and timestamps! Wait... it doesn't need to be online, you say? Are you telling me there is a command-line Linux program that can already do the same thing? Well, of course, we already knew that! Our website actually just passes your input right along to that command-line program!

### Find the flag in this vulnerable web application!

_The developer likes to store data in environment variables, can you find anything of interest there?_

1. In the `http://<TARGET_IP>` search bar type:

```bash
; env
```

[SCREEN01]
