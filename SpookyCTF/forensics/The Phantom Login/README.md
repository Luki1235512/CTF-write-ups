# [The Phantom Login](https://spooky.ctfd.io/challenges#The%20Phantom%20Login-2)

It's Halloween night at NJIT, and something strange has been happening on the network.

At 3:13 AM, the university's security team detected unusual authentication activity involving an account that should have been forgotten long ago. The SOC recovered Windows security logs and web access records, but the evidence is buried among hundreds of seemingly normal events. Your task is to investigate the suspicious activity, determine what happened, and uncover the secret left behind. The ghosts of NJIT don't give up their secrets easily.

1. Open `njit_web.log`. Almost every line comes from internal `10.13.x.x` addresses. Filter for anything external:

```bash
grep -v '10\.13\.' njit_web.log
```

2. This returns the only external source, `203.0.113.77`, logged in as `evelyn.graves`:

```
2026-10-31T03:16:10Z src=203.0.113.77 user=evelyn.graves "GET /archives/ HTTP/1.1" status=200
2026-10-31T03:18:20Z src=203.0.113.77 user=evelyn.graves "GET /archives/records?year=1893 HTTP/1.1" status=200
2026-10-31T03:19:08Z src=203.0.113.77 user=evelyn.graves "GET /archives/export?note=c3Bvb2t5e3RoZV9naG9zdF9sZWZ0X2FfdHJhY2V9 HTTP/1.1" status=200
```

3. The `note` parameter on the export request looks like base64. Decode it:

```bash
echo 'c3Bvb2t5e3RoZV9naG9zdF9sZWZ0X2FfdHJhY2V9' | base64 -d
```

[SCREEN01]
