---
title: Brecha en la nube - PwnedLabs
date: 2024-11-29
description: Usando los logs de AWS CloudTrail para detectar actividad maliciosa.
isStarred: true
---


![Intro](intro.jpg)

Este laboratorio muestra el uso de registros de AWS CloudTrail para detectar actividad maliciosa, así como la enumeración de S3. 
Para comenzar, descargue los registros de CloudTrail en INCIDENT-3252.zip del canal 🔎-case-files en Pwned Labs Discord: [https://discord.gg/pwnedlabs](https://discord.gg/pwnedlabs)

## Escenario

Hemos sido alertados de un posible incidente de seguridad. El equipo de seguridad de Huge Logistics le ha proporcionado claves de AWS de una cuenta que registró actividad inusual, así como registros de AWS CloudTrail en el momento de la actividad. Necesitamos su experiencia para confirmar la infracción analizando nuestros registros de CloudTrail, identificando el servicio de AWS comprometido y cualquier dato que haya sido filtrado. 

# Contexto del mundo real

El análisis de los registros de AWS CloudTrail es una práctica estándar para detectar actividad sospechosa dentro de una cuenta de AWS, mientras que los atacantes suelen atacar los depósitos S3 debido a los datos valiosos que pueden contener.

## **Tutorial**

Para comenzar, descargue los registros de CloudTrail `INCIDENT-3252.zip` del canal case-files en Pwned Labs [Discord](https://discord.gg/pwnedlabs)  .
Con el expediente del caso descargado, comencemos la investigación.

Primero descomprima el archivo con `unzip`. 

```
unzip INCIDENT-3252.zip -d INCIDENT-3252
cd INCIDENT-3252
```

![image.png](image.png)

Abrir el archivo en un editor de texto como Nano o Vim revela que los
 archivos JSON no están embellecidos, lo que los hace más difíciles de 
leer. Para embellecer los archivos podemos usar `jq` - un procesador JSON de línea de comandos para analizar y estructurar datos. Instálelo si es necesario con el comando `apt install jq` . Luego ejecute el siguiente comando en el directorio actual que contiene los archivos JSON.

![image.png](image%201.png)

```bash
for file in *.json; do jq . "$file" > "$file.tmp" && mv "$file.tmp" "$file"; done
```

Al abrir el archivo en nano, los datos ahora son mucho más fáciles de leer y escanear. 

![image.png](image%202.png)

Veamos qué AWS Principals (usuarios y roles de IAM) han estado generando actividad en los registros capturados. 

![image.png](image%203.png)

Esto revela el nombre de usuario sospechoso, `temp-user` , que no coincide con la convención de nomenclatura interna para cuentas creadas. Empecemos por ahí. 

Notamos que los registros de CloudTrail contienen la hora de creación en
 el nombre del archivo y están ordenados por hora. El momento más 
temprano es `T2035` así que comencemos por ahí. Sospechamos que el `temp-user` cuenta está involucrada, y grep el archivo con `temp-user` como término de búsqueda. 

```bash
grep -h -A 10 temp-user 107513503799_CloudTrail_us-east-1_20230826T2035Z_PjmwM7E4hZ6897Aq.json
```

![image.png](image%204.png)

Esto revela que un usuario de IAM llamado `temp-user` con el ARN (Amazon Resource Name) único a nivel mundial `arn:aws:iam::107513503799:user/temp-user` desde la cuenta de AWS `107513503799` emitió el comando CLI `aws sts get-caller-identity` en `2023-08-26T20:29:37Z` . 

La acción `GetCallerIdentity` en AWS es parte del [Security Token Service (STS)](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html) y permite a los usuarios recuperar detalles sobre la identidad de IAM cuyas credenciales se utilizan para realizar la solicitud de API. Se puede considerar un poco como el comando `whoami` para Windows y Linux. El comando devuelve el ARN globalmente único y, si corresponde, el ARN del rol de IAM asumido. Si bien este es un comando de uso común, también lo utilizan actores malintencionados para determinar el principal (usuario o rol de IAM) asociado con las credenciales comprometidas, como parte de la obtención de conocimiento 
de la situación. 

La solicitud se originó desde la dirección IP. `84.32.71.19` .Una verificación de la propiedad intelectual revela que la solicitud se originó en Chicago, US. Este no es un país donde Huge Logistics tenga presencia técnica, por lo que esto podría ser un posible indicador de compromiso (IoC). Profundicemos más. 

![image.png](image%205.png)

Volviendo nuestra atención al siguiente archivo, una mirada rápida revela que la cuenta `temp-user` hizo un intento fallido de enumerar el contenido de un depósito llamado `emergency-data-recovery` .

```bash
nano 107513503799_CloudTrail_us-east-1_20230826T2040Z_UkDeakooXR09uCBm.json
```

![image.png](image%206.png)

La investigación del siguiente archivo revela muchos errores. El valor `errorMessage` revela que `temp-user` emitió una gran cantidad de solicitudes a otros servicios de AWS. 

![image.png](image%207.png)

Aunque es muy ruidoso, los actores malintencionados pueden intentar 
enumerar los permisos otorgados a su usuario o función de IAM mediante 
fuerza bruta. Hay varias herramientas disponibles para forzar permisos 
de IAM por fuerza bruta, como  [aws-enumerator](https://github.com/shabarkin/aws-enumerator)  y  [pacu](https://github.com/RhinoSecurityLabs/pacu)  . Encontramos que hay 450 mensajes de acceso denegado generados por `temp-user` en el siguiente archivo de registro. 

![image.png](image%208.png)

Al analizar el siguiente archivo de registro y buscar nuevamente las acciones invocadas por el usuario, vemos que pudo asumir el rol denominado `AdminRole`. La operacion `AssumeRole` 
en AWS es parte de STS. Permite que una identidad de AWS asuma un contexto de privilegios diferente durante un período limitado y potencialmente acceda a otros recursos a los que el principal asumido normalmente no tendría acceso. 

![image.png](image%209.png)

El examen del siguiente archivo revela que el atacante invocó nuevamente el comando `aws sts get-caller-identity` para verificar su nuevo contexto de ejecución. 

```bash
grep -A 20 AdminRole 107513503799_CloudTrail_us-east-1_20230826T2105Z_fpp78PgremAcrW5c.json
```

![image.png](image%2010.png)

Tomando nota de su interés anterior en el bucket s3 `emergency-data-recovery` , descubrimos que volvieron a intentar enumerar y recuperar el contenido. 

```bash
grep eventName 107513503799_CloudTrail_us-east-1_20230826T2120Z_UCUhsJa0zoFY3ZO0.json
```

![image.png](image%2011.png)

```bash
grep -A 20 ListObjects 107513503799_CloudTrail_us-east-1_20230826T2120Z_UCUhsJa0zoFY3ZO0.json
```

![image.png](image%2012.png)

```bash
grep -A 20 GetObject 107513503799_CloudTrail_us-east-1_20230826T2120Z_UCUhsJa0zoFY3ZO0.json
```

![image.png](image%2013.png)

## **Volviendo sobre pasos como atacante**

Ahora, pensando como un atacante, queremos validar la ruta de explotación y acceder a los datos comprometidos. Emitmios el `aws configure` comando para configurar las claves de AWS proporcionadas en la parte superior del laboratorio para el usuario que creemos que está 
comprometido. A continuación confirmamos nuestro contexto de ejecución. 

![image.png](image%2014.png)

Veamos si tenemos políticas de usuario en línea adjuntas a nuestro usuario de IAM. 

```bash
aws iam list-user-policies --user-name temp-user --profile lab
```

![image.png](image%2015.png)

Esto revela la política denominada `test-temp-user`. Comprobémoslo. 

```bash
aws iam get-user-policy --user-name temp-user --policy-name test-temp-user
```

![image.png](image%2016.png)

Esto revela que el usuario puede asumir el rol denominado `AdminRole`.

Una vez identificado el role, el atacante intentó asumirlo. Podemos hacer esto con el siguiente comando.

![image.png](image%2017.png)

Para asumir el papel, necesitamos emitir el `aws configure` comando para utilizar el `AccessKeyId` y `SecretAccessKey` proporcionada en el paso anterior. Luego emita el comando `aws configure set aws_session_token "<SessionToken>"` para configurar el token. Una vez hecho esto podemos llamar al comando. `aws sts get-caller-identity` ¡Y verifica nuestro nuevo contexto! 

![image.png](image%2018.png)

Ahora podemos enumerar el contenido del depósito y recuperar los archivos. La bandera está en el archivo `emergency.txt`. 

```bash
aws s3 ls s3://emergency-data-recovery --profile lab
```

![image.png](image%2019.png)

```bash
aws s3 cp s3://emergency-data-recovery/emergency.txt . --profile lab
```

![image.png](image%2020.png)

---

## Referencias

- [Laboratorio PwnedLabs](https://pwnedlabs.io/labs/breach-in-the-cloud)
- [CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)
- [STS Api Reference](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html)