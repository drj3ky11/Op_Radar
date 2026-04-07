## Demo dificultad del hash

### 1. Obtener la cabecera (160 hex chars = 80 bytes)
```bash
HEADER=$(curl -s "https://mempool.space/api/block/HASH_DEL_BLOQUE/header")
```
### 2. Doble SHA-256 + inversión de bytes (big-endian para mostrar)
```bash
echo "$HEADER" \
  | xxd -r -p \
  | sha256sum | awk '{print $1}' \
  | xxd -r -p \
  | sha256sum | awk '{print $1}' \
  | fold -w2 | tac | tr -d '\n' && echo
```

### nonce 0  → little-endian "00000000"
```bash
HEADER_MOD="${HEADER:0:152}00000000"
echo "$HEADER_MOD" | xxd -r -p | sha256sum | awk '{print $1}' | xxd -r -p | sha256sum | awk '{print $1}' | fold -w2 | tac | tr -d '\n' && echo
```
### nonce 1  → little-endian "01000000"

```bash
HEADER_MOD="${HEADER:0:152}01000000"
echo "$HEADER_MOD" | xxd -r -p | sha256sum | awk '{print $1}' | xxd -r -p | sha256sum | awk '{print $1}' | fold -w2 | tac | tr -d '\n' && echo
```
## Powershell

### 1. Obtener cabecera

```Powershell
$H = (Invoke-WebRequest "https://mempool.space/api/block/HASH_DEL_BLOQUE/header").Content.Trim()
```

### 2. Hex → bytes

```Powershell
$bytes = [System.Convert]::FromHexString($H)   # PowerShell 7+
# Si PowerShell 5: $bytes = [byte[]]($H -split '(.{2})' | ?{$_} | %{[Convert]::ToByte($_,16)})
```

### 3. Doble SHA-256

```Powershell
$sha = [System.Security.Cryptography.SHA256]::Create()
$h1  = $sha.ComputeHash($bytes)
$h2  = $sha.ComputeHash($h1)
```

### 4. Invertir bytes y mostrar

```Powershell
[Array]::Reverse($h2)
($h2 | % { $_.ToString("x2") }) -join ''
```
