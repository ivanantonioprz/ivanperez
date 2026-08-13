# XOR Encryption & CyberChef

___

## 1. Objetivo

El objetivo de esta práctica fue analizar un programa en **Python** que implementa un mecanismo de cifrado basado en **XOR**, identificar cómo se genera la clave, obtener información mediante una conexión TCP y utilizar **CyberChef** para recuperar las claves y descifrar las cadenas obtenidas.

Durante la práctica se trabajó con:

- **XOR**
- **Codificación hexadecimal**
- **Claves repetitivas**
- **Known-Plaintext Attack**
- **CyberChef**
- **Netcat**
- **Python**
- **Kali Linux**

___

## 2. Análisis del código

El programa genera una clave aleatoria de 5 caracteres:

```python
res = ''.join(random.choices(string.ascii_letters + string.digits, k=5))
key = str(res)
```

La clave puede contener letras mayúsculas, minúsculas y números.

Posteriormente, se aplica XOR entre cada carácter de la flag y un carácter de la clave:

```python
for i in range(0, len(flag)):
    xored += chr(ord(flag[i]) ^ ord(key[i % len(key)]))
```

La expresión:

```python
key[i % len(key)]
```

hace que la clave se repita a lo largo del texto.

Finalmente, el resultado se convierte a hexadecimal:

```python
hex_encoded = xored.encode().hex()
```

### Flujo de cifrado

```text
FLAG
  ↓
XOR + KEY
  ↓
HEX Encode
  ↓
Encrypted Text
```

### Flujo de descifrado

```text
Encrypted HEX
  ↓
From Hex
  ↓
XOR + KEY
  ↓
Plaintext
```

___

## 3. Ejecución del servidor

Primero se creó el archivo necesario `flag.txt` para poder ejecutar el programa localmente.

El servidor se inició mediante:

```bash
python3 source-1705339805281.py
```

Inicialmente apareció el siguiente error:

```text
OSError: [Errno 98] Address already in use
```

Esto indicaba que el puerto `1337` ya estaba siendo utilizado.

Para identificar el proceso se utilizó:

```bash
sudo lsof -i :1337
```

Se encontró un proceso Python utilizando el puerto:

```text
python  24251  kali  ... TCP *:1337 (LISTEN)
```

El proceso fue terminado mediante:

```bash
sudo kill 24251
```

Posteriormente se comprobó nuevamente:

```bash
sudo lsof -i :1337
```

Al no obtener resultados, el puerto quedó disponible.

___

## 4. Conexión al servidor

Con el servidor ejecutándose en una terminal, se abrió una segunda terminal y se realizó una conexión mediante **Netcat**:

```bash
nc 127.0.0.1 1337
```

El servidor proporcionó la siguiente cadena hexadecimal:

```text
012c1a2b3a3d0d24393d3402363b2b3308363733
```

También solicitó la clave utilizada para el cifrado.

___

## 5. Descifrado de la primera cadena

Para descifrar la cadena se utilizó **CyberChef**.

La receta utilizada fue:

```text
From Hex
    ↓
XOR
```

Una de las primeras pruebas produjo:

```text
DHM{YZt|YcRs_|cMR`c
```

Esto permitió comprobar que la receta estaba funcionando, pero la clave todavía no era correcta.

### Known-Plaintext Attack

Se conoce que las flags de TryHackMe comienzan normalmente con:

```text
THM{
```

Debido a las propiedades de XOR:

```text
Ciphertext = Plaintext XOR Key
```

también podemos obtener:

```text
Key = Ciphertext XOR Plaintext
```

Utilizando los primeros cuatro caracteres conocidos:

```text
01 XOR T = U
2c XOR H = d
1a XOR M = W
2b XOR { = P
```

Se obtuvo:

```text
UdWP?
```

Después de determinar el quinto carácter se obtuvo la clave:

```text
UdWPN
```

Al introducir esta clave en CyberChef se obtuvo:

```text
THM{thisisafakeflag}
```

### Flag 1

```text
THM{thisisafakeflag}
```

### Clave 1

```text
UdWPN
```

___

## 6. Obtención de la segunda cadena

Después de proporcionar la clave correcta al servidor, este respondió con una segunda cadena cifrada:

```text
0d2a3a344368031b21471c1a030e472d56142450180c057c52352e0e27662b160e7f462b1a383d4e
```

Se utilizó nuevamente CyberChef:

```text
From Hex
    ↓
XOR
```

___

## 7. Recuperación de la segunda clave

Nuevamente se utilizó el prefijo conocido:

```text
THM{
```

Aplicando XOR entre los primeros bytes y los caracteres conocidos:

```text
0d XOR T = Y
2a XOR H = b
3a XOR M = w
34 XOR { = O
```

Se obtuvo:

```text
YbwO?
```

Después de determinar el quinto carácter se obtuvo:

```text
YbwO3
```

La clave fue introducida en CyberChef junto con la segunda cadena hexadecimal.

### Flag 2

```text
THM{p1alntExtAtt4ckcAnr3alLyhUrty0urxOr}
```

### Clave 2

```text
YbwO3
```

___

## 8. Herramientas utilizadas

### Python

Para ejecutar el código del servidor:

```bash
python3 source-1705339805281.py
```

### Netcat

Para conectarse al servidor TCP:

```bash
nc 127.0.0.1 1337
```

### lsof

Para identificar procesos utilizando el puerto `1337`:

```bash
sudo lsof -i :1337
```

### CyberChef

Se utilizó para:

1. Convertir hexadecimal mediante **From Hex**.
2. Aplicar **XOR**.
3. Probar las claves obtenidas.
4. Obtener las flags en texto plano.

___

## 9. Conceptos aprendidos

### XOR

XOR es una operación reversible:

```text
A XOR B = C
C XOR B = A
```

Por esta propiedad, el mismo algoritmo puede utilizarse para cifrar y descifrar si conocemos la clave.

### Hexadecimal

El resultado del XOR fue convertido a hexadecimal, por lo que fue necesario utilizar:

```text
From Hex
```

antes de aplicar XOR.

### Clave repetitiva

La clave tiene una longitud de 5 caracteres y se repite mediante:

```python
key[i % len(key)]
```

Por ejemplo:

```text
UdWPN
UdWPN
UdWPN
UdWPN
```

### Known-Plaintext Attack

El concepto principal utilizado fue un **Known-Plaintext Attack**.

Al conocer una parte del texto original:

```text
THM{
```

es posible obtener parte de la clave mediante:

```text
Ciphertext XOR Known Plaintext = Key
```

Esto permitió recuperar las claves utilizadas para descifrar las cadenas.

___

## 10. Resultados finales

| Elemento | Resultado |
|---|---|
| **Método** | XOR + Hexadecimal |
| **Longitud de clave** | 5 caracteres |
| **Clave 1** | `UdWPN` |
| **Flag 1** | `THM{thisisafakeflag}` |
| **Clave 2** | `YbwO3` |
| **Flag 2** | `THM{p1alntExtAtt4ckcAnr3alLyhUrty0urxOr}` |
| **Herramienta principal** | CyberChef |
| **Conexión** | Netcat |
| **Puerto** | 1337 |

___

## 11. Conclusión

Esta práctica permitió comprender cómo analizar y revertir un sistema sencillo de cifrado **XOR** cuando se utiliza una clave repetitiva.

El punto más importante fue reconocer que conocer el comienzo de una flag, en este caso:

```text
THM{
```

permite recuperar parte de la clave debido a que XOR es una operación reversible.

El flujo utilizado fue:

```text
Código Python
     ↓
Análisis del algoritmo
     ↓
Servidor TCP
     ↓
Netcat
     ↓
Cadena hexadecimal
     ↓
From Hex
     ↓
Known-Plaintext Attack
     ↓
Recuperación de la clave
     ↓
XOR en CyberChef
     ↓
Flag
```

La práctica permitió adquirir experiencia utilizando **CyberChef, Python, Netcat, Kali Linux y herramientas de análisis de procesos y puertos**, además de comprender cómo una implementación de XOR con una clave corta y repetitiva puede ser vulnerable a un **Known-Plaintext Attack**.