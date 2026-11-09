# [Scopophobia](https://spooky.ctfd.io/challenges#Scopophobia-44)

After months of unemployment, you finally land a big ol’ j-o-b in security. It’s minimum wage but hey, the experience is what matters. Though what they didn’t tell you was the physical security aspect. Analyze the dashboard and read the source to try to find what they don’t want you finding.

http://18.189.35.130/

## 1. Read the dashboard source

```bash
curl -s http://18.189.35.130/ | head -c 600
```

The first lines of the page contain two HTML comments:

```html
<!-- TODO: Legacy camera inventory was excluded from the current monitoring migration. Confirm whether archival feeds were properly retired. -->
<!-- CAM-07 was scheduled for decommissioning but remains registered. -->
```

The only camera link in the sidebar is `/feeds/active/cam-01`. Its `active` segment suggests a sibling namespace for retired cameras, and the comments name CAM-07 as the one that was never removed. The 404 page for a missing feed says "The requested camera channel is not registered", which gives a clear signal for what is and isn't a real route.

## 2. Find the legacy CAM-07 feed

```bash
curl -s -o /dev/null -w "%{http_code} %{size_download}\n" http://18.189.35.130/feeds/legacy/cam-07
```

This returns `200 2289`. A bad path returns `404 1312`. I got here by trying several namespace names (`archive`, `archived`, `legacy`, `retired`, `decommissioned`, `inactive`, `old`) with `cam-07`. Only `legacy/cam-07` returned 200. Paths are case sensitive, so `CAM-07` fails.

Fetch the page and read its comments:

```bash
curl -s http://18.189.35.130/feeds/legacy/cam-07 | grep -n -E '<!--|data-animation'
```

Output:

```html
<!-- Imaging migration record -->
<!-- Service camera directory: /feeds/service/ -->
... data-animation="/feeds/legacy/cam-07/frame" ...
```

The next namespace is `service`. The camera ID is not given yet.

## 3. Read the sign in the CAM-07 GIF

Download the GIF. Do not use `-i` here, because it writes the HTTP headers into the file and corrupts the image.

```bash
mkdir -p ~/CTF/cam07 && cd ~/CTF/cam07
curl -s http://18.189.35.130/feeds/legacy/cam-07/frame -o cam07.gif
file cam07.gif
convert cam07.gif -coalesce f_%03d.png
montage f_00*.png -tile 3x2 -geometry +4+4 sheet.png
display sheet.png
```

The GIF has 6 frames of an empty stairwell with a hand creeping over the railing. The wall sign reads `SERVICE CORRIDOR - SC-12`.

## 4. Open the service feed

The service directory does not use `cam-NN` names (I tried `cam-01` to `cam-10` and `cam-srv-NN`, all 404). Use the sign text in lowercase, matching the `cam-01` and `cam-07` pattern:

```bash
curl -s -o /dev/null -w "%{http_code} %{size_download}\n" http://18.189.35.130/feeds/service/sc-12
```

This returns `200 2307`. Check the comments and download the GIF:

```bash
curl -s http://18.189.35.130/feeds/service/sc-12 | grep -n -E '<!--|data-animation'
curl -s http://18.189.35.130/feeds/service/sc-12/frame -o sc12.gif
convert sc12.gif -coalesce g_%03d.png
montage g_00*.png -tile 4x2 -geometry 768x432+4+4 sheet12.png
display sheet12.png
```

The comments:

```html
<!-- Service feed migration incomplete -->
<!-- Redundant imaging namespace: /feeds/mirror/ -->
```

The GIF has 8 frames of an industrial corridor. The wall sign next to the doors reads `MIRROR DOWNLINK: CAM 19`.

## 5. Open the mirror feed

```bash
curl -s -o /dev/null -w "%{http_code} %{size_download}\n" http://18.189.35.130/feeds/mirror/cam-19
```

This returns `200 2459`. The last digit on the sign is blurry, and `cam-18` and `cam-13` return 404, so `cam-19` is correct.

```bash
curl -s http://18.189.35.130/feeds/mirror/cam-19 | grep -n -E '<!--|data-animation'
curl -s http://18.189.35.130/feeds/mirror/cam-19/frame -o mirror.gif
convert mirror.gif -coalesce m_%03d.png
montage m_00*.png -tile 5x2 -geometry 600x338+4+4 sheet19.png
display sheet19.png
```

The comments:

```html
<!-- Restricted camera migration -->
<!-- Sublevel imaging uses a three-letter directory alongside the other feeds. -->
```

The directory name is three letters and relates to "sublevel", so it is `sub`. The GIF has 10 frames of a network operations room. A small sign at the top left of the room reads `SUB LEVEL B - SLB-04`. It is tiny, so open a single frame (for example `display m_007.png`) and zoom in. The sign code is the camera ID.

## 6. Open the sublevel feed and read the flag

```bash
curl -s -o /dev/null -w "%{http_code} %{size_download}\n" http://18.189.35.130/feeds/sub/slb-04
```

This returns `200 2215`. The directory is `sub` and the ID is `slb-04`. Using `slb` as the directory returns 404, and so do `SLB-04` and `slb04`.

```bash
curl -s http://18.189.35.130/feeds/sub/slb-04 | grep -n -E 'data-timed-reveal|data-animation'
curl -s http://18.189.35.130/feeds/sub/slb-04/frame -o slb04.gif
convert slb04.gif -coalesce s_%03d.png
montage s_00*.png -tile 4x2 -geometry 768x432+4+4 sheet04.png
display sheet04.png
```

The `img` tag has the `data-timed-reveal` attribute. In `app.js` that attribute makes the player hold the second frame (the cryptid face) forever after 3.5 seconds. The GIF has 7 frames. The room has a locked door marked `RESTRICTED ENTRY`, and hazard tape on the floor holds the flag text in every frame:

<img width="1954" height="901" alt="screen01" src="https://github.com/user-attachments/assets/26680306-ad95-4779-b0d0-c1dd22360c20" />
