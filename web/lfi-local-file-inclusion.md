# 📄 LFI — Local File Inclusion

**Categoría:** Web Application Pentesting  
**Dificultad:** Principiante  
**Relevancia eJPT:** ✅ Web Application Penetration Testing  
**Fecha:** 2026-03-20

---

## 🧠 ¿Qué es LFI?

LFI (Local File Inclusion) es una vulnerabilidad web que ocurre cuando una aplicación carga archivos del servidor basándose directamente en lo que el usuario le pasa por la URL — sin validar ni restringir qué archivos puede pedir.

### La analogía del archivador

Imagina un empleado de oficina al que le dicen:
> *"Cuando alguien te pida un documento, ve al archivador y tráeselo."*

El empleado es obediente y no tiene instrucciones de rechazar ninguna petición. Si alguien le pide las nóminas secretas de todos los empleados... también se las da.

**Eso es exactamente LFI:** el servidor es el empleado obediente, el sistema de archivos es el archivador, y el atacante es quien pide documentos que no debería ver.

---

## 🔍 ¿Cómo se ve en el código?

Un programador quiere cargar páginas dinámicamente:

```php
<?php
  $pagina = $_GET['page'];
  include($pagina . ".html");
?>
```

**URL normal:**
```
http://victima.com/index.php?page=contacto
→ carga contacto.html ✅
```

**URL maliciosa:**
```
http://victima.com/index.php?page=../../../../etc/passwd
→ carga /etc/passwd ⚠️
```

Los `../` significan "sube un nivel en las carpetas" — igual que `cd ..` en Linux.

---

## 🎯 ¿Qué archivos buscar?

### `/etc/passwd` — el primero siempre
No contiene contraseñas directamente (eso está en `/etc/shadow`), pero sí contiene:

```
root:x:0:0:root:/root:/bin/bash
debian:x:1001:1001::/home/debian:/bin/bash
www-data:x:33:33::/var/www:/bin/sh
```

**Formato:** `usuario:x:UID:GID:descripción:home:shell`

**¿Por qué es valioso?**
- 📋 Lista todos los usuarios del sistema
- 🔍 La shell al final dice si el usuario puede conectarse (`/bin/bash` = sí, `/sbin/nologin` = no)
- ✅ Confirma que el LFI funciona (está en todos los sistemas Linux)

### Otros archivos interesantes
| Archivo | ¿Qué contiene? |
|---|---|
| `/etc/passwd` | Usuarios del sistema |
| `/etc/shadow` | Hashes de contraseñas (solo root puede leerlo) |
| `/var/www/html/config.php` | Credenciales de base de datos |
| `app.py` / `index.php` | Código fuente de la aplicación |

---

## 🔓 Bypassear filtros

Los programadores a veces intentan "arreglar" la vulnerabilidad eliminando los `../`:

```php
$pagina = str_replace("../", "", $_GET['page']);
```

### ¿Cómo saltarlo?

El filtro borra `../` una sola vez sin revisar el resultado. Solución:

```
Escribes:   ....//....//....//etc/passwd
El filtro borra "../" del medio:
Queda:      ../../../etc/passwd  ✅
```

**Principio clave:** Antes de bypassear un filtro, entiende exactamente qué hace y dónde está su límite.

---

## 🔗 LFI encadenado — caso real (Máquina Relojería)

LFI raramente actúa solo. Ejemplo real de cómo se encadena:

```
Flask con modo debug activado
        ↓
Debug expone una consola en /console
        ↓
La consola tiene un PIN de seguridad
        ↓
LFI nos permite leer app.py
        ↓
app.py tiene el PIN hardcodeado
        ↓
Accedemos a /console con el PIN
        ↓
Ejecutamos código Python en el servidor
        ↓
Reverse Shell ✅
```

> 💡 Esto se llama **encadenar vulnerabilidades** — ninguna sola era suficiente, pero juntas daban acceso total.

---

## 🛠️ Herramientas usadas con LFI

| Herramienta | ¿Para qué? |
|---|---|
| `ffuf` / `gobuster` | Encontrar parámetros vulnerables y directorios |
| `wfuzz` | Fuzzing con filtrado de ruido (`--hw`) |
| Browser / curl | Probar el payload manualmente |
| Hydra | Fuerza bruta una vez obtenidos los usuarios |

---

## ✅ Checklist LFI

```
[ ] Identificar parámetros en la URL (?page=, ?file=, ?id=)
[ ] Probar ../../../../etc/passwd
[ ] Si hay filtro, intentar bypass (....// u otros)
[ ] Leer /etc/passwd para obtener usuarios
[ ] Identificar usuarios con /bin/bash
[ ] Buscar archivos de configuración o código fuente
[ ] Encadenar con el siguiente paso del ataque
```

---

## 📚 Máquinas donde apliqué esto

| Máquina | Plataforma | Técnica usada |
|---|---|---|
| LavaShop | The Hackers Labs | LFI → /etc/passwd → Hydra SSH |
| Relojería | The Hackers Labs | LFI → app.py → Werkzeug PIN → RCE |

---

## 🔗 Referencias

- [OWASP — LFI](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion)
- [PayloadsAllTheThings — LFI](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion)
