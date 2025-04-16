# [Ninja Skills](https://tryhackme.com/room/ninjaskills)

## Practise your Linux skills and complete the challenges.

# Ninja Skills

## Let's have some fun with Linux. The aim is to answer the questions as efficiently as possible.

### Which of the above files are owned by the best-group group(enter the answer separated by spaces in alphabetical order)

```bash
find / -type f -group best-group 2>>/dev/null
```

- `find /`: Starts a search from the root director
- `-type f`: Looks for regular files
- `-group best-group`: Finds files owned by the group "best-group"
- `2>>/dev/null`: Redirects error messages to /dev/null, which suppresses permission denied errors

![SCREEN01](https://github.com/user-attachments/assets/829d8d7c-1a85-41e5-936b-89d4b6ed8801)

### Which of these files contain an IP address?

```bash
find / -type f -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o -name FHl1 -o -name oiMO -o -name PFbD -o -name rmfX -o -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy 2>/dev/null | xargs grep -l -P '(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)'
```

- First part searches for specific named files
- `|`: Pipes the found file paths to the next command
- `xargs`: Takes the file paths and passes them as arguments to grep
- `grep -l`: Lists only the file names that contain matches
- `-P '...'`: Regex to match IPv4 addresses
- The regex pattern matches valid IP address format (0-255.0-255.0-255.0-255)

![SCREEN02](https://github.com/user-attachments/assets/91d82385-1560-4571-ad4a-21deb9d046e7)

### Which file has the SHA1 hash of 9d54da7584015647ba052173b84d45e8007eba94

```bash
find / -type f \( -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o -name FHl1 -o -name oiMO -o -name PFbD -o -name rmfX -o -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \) -exec sh -c 'sha1sum "$1" | grep -q "9d54da7584015647ba052173b84d45e8007eba94" && echo "$1"' _ {} \; 2>/dev/null
```

- `\( ... \)`: Groups the name conditions
- `-exec`: Executes a command on each found file
- `sh -c '...'`: Runs a shell command
- `sha1sum "$1"`: Calculates the SHA1 hash of each file
- `| grep -q "hash"`: Silently checks if the output contains the specific hash
- `&& echo "$1"`: If the hash matches, prints the filename
- `_ {}`: Passes the found file as argument to the shell command
- `\;`: Terminates the -exec command

![SCREEN03](https://github.com/user-attachments/assets/15659225-d181-4319-a3e6-30a467ec39c7)

### Which file contains 230 lines?

```bash
find / -type f \( -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o -name FHl1 -o -name oiMO -o -name PFbD -o -name rmfX -o -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \) -exec wc -l {} \; 2>/dev/null
```

- `-exec wc -l {} \`: Executes the `wc -l` command (word count, lines only) on each found file
- You would then need to look through the output to find the file that is not displayed

![SCREEN04](https://github.com/user-attachments/assets/11f01fd6-bc7f-4d22-b11d-1e23a53ad013)

### Which file's owner has an ID of 502?

```bash
find / -type f \( -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o -name FHl1 -o -name oiMO -o -name PFbD -o -name rmfX -o -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \) -uid 502 2>/dev/null
```

- `-uid 502`: Finds files whose owner has the user ID of 502

![SCREEN05](https://github.com/user-attachments/assets/a0c017fd-4bda-4d11-a940-64a9de3d233a)

### Which file is executable by everyone?

```bash
find / -type f \( -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o -name FHl1 -o -name oiMO -o -name PFbD -o -name rmfX -o -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \) -perm -a=x 2>/dev/null
```

- `-perm -a=x`: Finds files with the execute permission set for all users

![SCREEN06](https://github.com/user-attachments/assets/166a515c-c100-4969-bbd5-1b26c9ed793f)
