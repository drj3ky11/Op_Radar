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

Imprime los 160 caracteres hexadecimales de la cabecera (80 bytes del bloque en texto hex).
Convierte hex → bytes binarios. xxd -r = "reverso" (de hex a binario); -p = formato plano (sin direcciones). Resultado: 80 bytes en crudo.
Calcula el primer SHA-256 de esos 80 bytes. Devuelve 64 hex chars. El awk descarta el  *- que añade sha256sum al final.
Convierte esos 64 hex chars de nuevo a 32 bytes binarios (el digest del primer SHA-256).
Calcula el segundo SHA-256 → doble SHA-256 completo. Resultado: 64 hex chars.
Invierte el orden de los bytes para mostrarlo en big-endian (como lo muestra el explorador). fold -w2 parte en pares (1 byte = 2 hex chars, cada uno en su línea), tac invierte las líneas, tr -d '\n' las une.


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

## Extraer y expandir el target (campo bits)

El campo bits son los bytes 72-75 del header (posición 144-151 en hex). Está en little-endian y es una representación compacta del target de 256 bits.

```bash
# 1. Extraer bits del header (LE → BE)
BITS=$(echo "${HEADER:144:8}" | fold -w2 | tac | tr -d '\n')

# 2. Descomponer: 1er byte = exponente, los 3 siguientes = coeficiente
EXP=$((16#${BITS:0:2}))
COEF="${BITS:2:6}"

# 3. Construir el target completo de 64 hex chars (32 bytes)
LEAD=$(( (32 - EXP) * 2 ))
TRAIL=$(( (EXP - 3) * 2 ))
TARGET=$(printf '0%.0s' $(seq 1 $LEAD))${COEF}$(printf '0%.0s' $(seq 1 $TRAIL))

echo "Target : $TARGET"
echo "Hash   : $HASH"
```

Ejemplo de salida con un bloque reciente:

Target : 00000000000000000005389400000000000000000000000000000000000000000

Hash   : 00000000000000000000a7f0bbd5d8e380bd4503d923a5bf706ed59484943225

La regla es simple: el hash tiene que ser numéricamente menor que el target. Como ambos son hex big-endian, basta mirar de izquierda a derecha — el hash válido tiene más ceros iniciales que el target.


El campo bits = 0x17053894 (ejemplo) se decodifica así:

Exponente 0x17 = 23 El target ocupa 23 bytes

Coeficiente 053894 Los 3 bytes significativos

```
Target = 053894 × 256^(23-3)  →  000000000000000000 053894 00000000000000000000000000000000000000000000000000000000
         ← 9 bytes de ceros →  ← 3 bytes →  ← 20 bytes de ceros →
```
