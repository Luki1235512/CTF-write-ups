# [purgatory](https://nnsc.tf/challenges?challenge=rev_purgatory)

**Details:**
We hot patched the validator during business hours. Nothing crashed, which is a good sign.

## 1. Look at what you're given

The archive unpacks to this:

```
rev_purgatory/
  run.sh
  compose.yaml
  Dockerfile
  entrypoint.sh
  purgatory_runner.erl
  releases/
    old.beam
    new.beam
```

Read `purgatory_runner.erl` and `run.sh` first, before touching the beam files. The runner does exactly this:

1. Load `old.beam` as the module `purgatory`.
2. Call `purgatory:boot()`. This spawns a worker process and blocks until the worker confirms it's alive.
3. Load `new.beam` as `purgatory`, replacing the running code.
4. Read a passphrase from the socket, send it to the worker process as `{check, self(), Passphrase}`, and wait for `{verdict, Bool}`.

So there's a live worker process that was created while `old.beam` was the loaded code, and by the time it actually validates anything, `new.beam` has already replaced it. That's the whole premise of the challenge: figure out exactly what "replacing the code" does to a process that was already running.

## 2. Try to load the beam files, and hit a version wall

Point any Erlang shell you have at these files:

```erlang
{ok, Bin} = file:read_file("old.beam"),
code:load_binary(purgatory, "old.beam", Bin).
```

On a stock Ubuntu/Debian install this fails with something like:

```
{error,{features_not_allowed,[maybe_expr]}}
```

That error is a red herring in terms of content, but it tells you the real problem: these `.beam` files were compiled by a newer OTP whose on-disk chunk formats changed. You need a matching toolchain before you can do anything useful. Don't try to hand-parse the BEAM chunk format yourself, that's a waste of time when the real fix is just running the right VM.

## 3. Get an OTP 28 toolchain

Any of these works, pick whichever is fastest for your setup:

- `docker pull erlang:28-alpine` and work inside it.
- `kerl build 28.0 28.0 && kerl install 28.0 ~/otp-28` if you use kerl.
- `asdf install erlang 28.0` if you use asdf.
- Build from source if none of the above are available:

```bash
curl -LO https://github.com/erlang/otp/releases/download/OTP-28.0/otp_src_28.0.tar.gz
tar xzf otp_src_28.0.tar.gz
cd otp_src_28.0
./configure --without-wx --without-odbc --without-observer --without-et \
            --without-debugger --without-megaco --without-diameter \
            --without-ssh --without-jinterface --without-common_test \
            --without-eunit --without-dialyzer --without-hipe
make -j$(nproc)
export PATH="$PWD/bin:$PATH"
```

Once it's done, confirm it works:

```bash
erl -eval 'io:format("~s~n",[erlang:system_info(otp_release)]), halt().' -noshell
```

## 4. Diff old.beam and new.beam

Now that we have a real OTP 28, we can use the compiler's own disassembler instead of writing anything custom:

```erlang
BF = beam_disasm:file("old.beam"),
io:format("~p~n",[BF]),
halt().
```

Do the same for `new.beam`, save both outputs to files, and diff them. The result is short: the two files are byte-for-byte identical except for three embedded literal lists. Everything else is the same. This confirms the flavor text literally: this was a hot code reload that only swapped constant data, not logic.

## 5. Read the logic

`beam_disasm` gives us readable instructions with resolved names, so we can read the module directly instead of guessing at opcodes. The relevant functions:

```erlang
loop() ->
    receive
        {check, From, Bin} ->
            From ! {verdict, validate(Bin)},
            loop()
    end.

validate(Bin) when is_binary(Bin), byte_size(Bin) =:= 25 ->
    case first_half(Bin) of
        true  -> second_half(Bin, fun mask/2);
        false -> false
    end;
validate(_) -> false.

first_half(Bin) ->
    Table = [{0,247,40,120}, {1,195,0,68}, ... {12,69,35,108}],   %% 13 entries, bytes 0..12
    check(Table, fun(I) -> binary:at(Bin, I) end).

second_half(Bin, MaskFun) when is_binary(Bin), byte_size(Bin) =:= 25 ->
    Table = [{13,109,36,77}, {14,219,239,57}, ... {24,37,35,8}],  %% 12 entries, bytes 13..24
    check(Table, fun(I) -> MaskFun(I, binary:at(Bin, I)) end).

check(Table, AtFun) ->
    lists:all(fun({Idx, A, B, C}) ->
        Byte = AtFun(Idx),
        (A * Byte + B) band 255 =:= C
    end, Table).

mask(Idx, Byte) ->
    MaskTable = [114,157,234,123,50,120,87,159,59,45,110,181],   %% 12 bytes
    Byte bxor lists:nth(Idx - 12, MaskTable).
```

You can read this straight off the `beam_disasm` output: `get_tuple_element` pulls `Idx/A/B/C` out of each tuple, `*`, `+`, `band 255`, `=:=` build the comparison, and `mask/2` is just a subtraction, an `lists:nth` lookup, and an `bxor`.

So each byte of the 25-byte passphrase has to satisfy an affine check mod 256: `(A * byte + B) mod 256 == C`. Since every `A` here is odd, it's invertible mod 256, so each byte has exactly one valid value: `byte = (C - B) * modinv(A, 256) mod 256`. The second half additionally XORs the raw byte against a 12-entry mask table before the check runs.

## 6. Spot the actual trap

Look at how `validate` calls the two halves, this is the part that matters:

```erlang
{call,      1, {purgatory,first_half,1}}            % first_half: plain local call
...
{call_ext_last, 2, {extfunc,purgatory,second_half,2}, 1}   % second_half: fully-qualified call
```

That distinction is everything. In Erlang, a process that's mid-execution in one version of a module keeps running that version for as long as it only makes plain, unqualified calls to functions in the same module. It only switches to whatever code is "current" when it makes a fully-qualified call, an `extfunc` call like `purgatory:second_half(...)` here.

The worker process was created while `old.beam` was current, and from `loop()` down through `validate` and `first_half` it never leaves local calls, so all of that runs forever on `old.beam`, table and all, no matter what gets loaded later.

`second_half` is called via `extfunc`, so once the process reaches that call it switches to whatever is current at that moment, which is `new.beam`. So `second_half`'s own logic, its own 12-entry table, and its own local call to `check/2`, all run as `new.beam`.

But look closer at where the mask function comes from:

```erlang
{make_fun3,{purgatory,mask,2}, 0, 114435554, {x,1}, {list,[]}},
{move,{y,0},{x,0}},
{call_ext_last,2,{extfunc,purgatory,second_half,2},1}
```

`validate` builds the `fun mask/2` closure _before_ it calls `second_half`, and at that point it's still running old code. A fun value in Erlang is tied to the exact compiled version it was created from. So this particular `mask` closure stays pinned to `old.beam` permanently, even though it gets handed off into, and called from, code that's now running as `new.beam`.

Net result: three different data sources feed into one check.

- bytes 0-12: `old.beam`'s table
- bytes 13-24: `new.beam`'s table, but XORed against `old.beam`'s mask table

That's not something you want to talk yourself into by pure reasoning about VM internals. Verify it.

## 7. Verify it by actually running the scenario

Reproduce exactly what the runner does, then trace the worker process to see, in practice, which literal values get used at each step:

```erlang
{ok,OldBin} = file:read_file("old.beam"),
{module,purgatory} = code:load_binary(purgatory, "old.beam", OldBin),
Worker = purgatory:boot(),
{ok,NewBin} = file:read_file("new.beam"),
{module,purgatory} = code:load_binary(purgatory, "new.beam", NewBin),

erlang:trace(Worker, true, [call, return_to]),
erlang:trace_pattern({purgatory,'_','_'}, [{'_',[],[{return_trace}]}], [local]),
erlang:trace_pattern({lists,nth,2}, [{'_',[],[{return_trace}]}], [global]),

Worker ! {check, self(), <<"any 25 byte binary here!">>},
%% then just drain and print whatever trace messages come in
```

The trace output answers the question directly: the closure passed into `second_half` shows up as `#Fun<purgatory.0.114435554>`, the table used inside `check/2` for the second half matches `new.beam`'s literal, and `lists:nth` gets called against `old.beam`'s 12-byte mask list, not `new.beam`'s. That confirms the mixed-version theory.

This step is worth doing even if you're confident in your reasoning, because this exact module also has four decoy answers baked in on purpose. Guessing from readability alone will lead you to a wrong but very convincing-looking passphrase.

## 8. Solve for the passphrase

```python
def modinv(a, m=256):
    old_r, r = a % m, m
    old_s, s = 1, 0
    while r != 0:
        q = old_r // r
        old_r, r = r, old_r - q * r
        old_s, s = s, old_s - q * s
    return old_s % m  # old_r will be 1 here since every A used is odd

TableA_old = [(0,247,40,120), (1,195,0,68), (2,89,105,45), (3,77,179,70),
              (4,107,221,62), (5,37,117,101), (6,11,232,52), (7,205,46,5),
              (8,93,161,36), (9,47,220,181), (10,195,222,49), (11,191,233,251),
              (12,69,35,108)]

TableB_new = [(13,87,33,108), (14,233,61,201), (15,197,97,245), (16,155,124,72),
              (17,75,219,93), (18,115,213,5), (19,219,87,227), (20,221,230,166),
              (21,69,83,123), (22,139,156,253), (23,193,229,111), (24,93,241,181)]

MaskTable_old = [114,157,234,123,50,120,87,159,59,45,110,181]

result = [None] * 25

for idx, A, B, C in TableA_old:
    result[idx] = ((C - B) * modinv(A)) % 256

for idx, A, B, C in TableB_new:
    masked = ((C - B) * modinv(A)) % 256
    result[idx] = masked ^ MaskTable_old[idx - 13]

print(bytes(result))
```

This prints:

```
b'0ld_c0d3_w1n5_1n_th3_3nd!'
```

25 bytes, matches the `byte_size(Bin) =:= 25` guard, and reads as plain text, which lines up with the theme of the bug.

## 9. Confirm it locally before touching the remote server

Since we already have OTP 28 running and the exact files, replay the runner's own sequence one more time with this candidate:

```erlang
{ok,OldBin} = file:read_file("old.beam"),
{module,purgatory} = code:load_binary(purgatory, "old.beam", OldBin),
Worker = purgatory:boot(),
{ok,NewBin} = file:read_file("new.beam"),
{module,purgatory} = code:load_binary(purgatory, "new.beam", NewBin),
Worker ! {check, self(), <<"0ld_c0d3_w1n5_1n_th3_3nd!">>},
receive {verdict, V} -> io:format("~p~n",[V]) after 3000 -> io:format("timeout~n") end.
```

This should print `true`. That's our confirmation.

## 10. Get the flag

Connect to the service and send the passphrase:

```bash
ncat --ssl TARGET_IP 1337
passphrase> 0ld_c0d3_w1n5_1n_th3_3nd!
```

The service checks it against the same worker process logic we just verified locally, prints `NNS{...}`, and that's the flag.

[SCREEN01]
