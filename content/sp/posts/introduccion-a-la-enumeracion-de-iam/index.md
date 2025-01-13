---
title: Intro a la enumeracion de AWS IAM - PwnedLabs
date: 2024-11-29
description: Una introducción a la AWS CLI, así como a la enumeración de usuarios, roles, grupos y políticas de IAM.
isStarred: false
---

![aws_iam_enum.png](aws_iam_enum.png)

Una introducción a la AWS CLI, así como a la enumeración de usuarios, roles, grupos y políticas de IAM.

## [Laboratorio en PwnedLabs](https://pwnedlabs.io/labs/intro-to-aws-iam-enumeration)

## Escenario

Usted es un consultor de seguridad contratado por la empresa de logística global Huge Logistics. Después de una actividad sospechosa, usted tiene la tarea de enumerar el usuario de IAM `dev01` y mapear cualquier recurso potencialmente comprometido. Su misión es enumerar y evaluar roles, políticas y permisos de IAM. 

## Contexto del mundo real

IAM (Identity and Access Managment) es fundamental para crear, defender y atacar los servicios en la nube. Tanto los profesionales de la seguridad ofensiva como la defensiva necesitan una sólida comprensión de IAM y de cómo enumerar los permisos: los atacantes buscan configuraciones excesivamente permisivas o configuraciones erróneas en una posible killchain, mientras que los defensores se aseguran de la necesidad de hacer cumplir el principio de privilegio mínimo e identificar cualquier recurso o servicio que se encuentre en el radio de explosión de un usuario de IAM comprometido.

# Tutorial
### ¿Cómo podemos interactuar con AWS?

1. `Consola de administración de AWS`  : una interfaz web que proporciona una forma gráfica de administrar los recursos de AWS.
2. `AWS CLI` : la interfaz de línea de comandos le permite administrar los servicios de AWS a través de la línea de comandos.
3. `SDK`  : los kits de desarrollo de software están
disponibles en varios lenguajes como Python (Boto3), Java, .NET y más,
lo que permite el acceso programático a los servicios de AWS.
4. `API` : la mayoría de los servicios de AWS exponen API RESTful con las que puede interactuar directamente.

### Consola de administración de AWS

La URL de la Consola de administración de AWS para usuarios de IAM se parece al ejemplo siguiente. Nuestro ID de cuenta de AWS `794929857501` forma parte de la URL y AWS la utiliza para completar automáticamente el campo ID de cuenta de la página de inicio de sesión del usuario de IAM. Continúe, haga clic en el enlace e inicie sesión con las credenciales de usuario de AWS IAM proporcionadas.

![image.png](image.png)

Al iniciar sesión, vemos una colección de servicios visitados recientemente. En la esquina superior izquierda de la ventana podemos hacer clic en `All services` y desplácese por una lista alfabética de servicios. 

![image.png](image%201.png)

La consola de AWS ofrece una manera conveniente de acceder, crear, modificar y eliminar recursos.

Haga clic en `GuardDuty`b, en la lista alfabética o ábralo después de buscar el servicio. En la consola de GuardDuty que se abre, seleccione la región `us-west-1` de la lista desplegable.

![image.png](image%202.png)

Aunque la Consola de Administración de AWS es muy conveniente, la AWS CLI es muy poderosa y nos permite procesar comandos en lote y llevar a cabo automatizaciones, facilitando nuestras tareas.

Este laboratorio se centrará en la AWS CLI, ya que buscamos identificar el radio de impacto del usuario IAM potencialmente comprometido y lo que podría haber accedido.

Instalar la AWS CLI:

```csharp
sudo apt install awscli
```

En la CLI de AWS, nos autenticamos mediante la AWS Access Key y Secret Key.

La clave de acceso y la clave de acceso secreta juntas forman un conjunto de credenciales que se utilizan para autenticar solicitudes de API y funcionan como una especie de "nombre de usuario y contraseña". Estas claves se generan a través de la Consola de administración de AWS y 
están vinculadas a un usuario de IAM o una cuenta raíz de AWS.

1. `Access Key ID`: Es una cadena alfanumérica de 20 caracteres (por ejemplo, "AKIAIOSFODNN7EXAMPLE"). Se utiliza para identificar al usuario o la cuenta que realiza una solicitud programática a un servicio de AWS. Está destinado a ser compartido, como un nombre de usuario.
2. `Secret Access Key`: Es una cadena de 40 caracteres (por ejemplo, "wJalrXUtnFEMI/K7MEXAMPLEKEYEXAMPLEKEY") y sirve como "contraseña" para autenticar las solicitudes realizadas con el ID de clave de acceso correspondiente. A diferencia del ID de la clave de acceso, la clave de acceso secreta es confidencial y nunca debe compartirse ni almacenarse de manera insegura.

Podemos configurar las claves con el comando AWS CLI aws configure, como se muestra a continuación.

```csharp
aws configure --profile lab
```

![image.png](image%203.png)

Una vez configuradas las claves podremos interactuar con AWS en el contexto de ejecución de este usuario. 

## Comenzando la enumeracion

El siguiente comando puede considerarse como el comando `whoami` en Windows y Linux. Como era de esperar, esto revela que estamos en el contexto del usuario de IAM llamado `dev01` en la cuenta de AWS `794929857501` . 

```python
aws sts get-caller-identity --profile lab
```

![image.png](image%204.png)

Obtengamos más información sobre nuestro usuario potencialmente comprometido.

```csharp
aws iam get-user --profile lab
```

![image.png](image%205.png)

Esto también proporciona la fecha en que se creó el usuario de IAM y la etiqueta. Una etiqueta es un par clave-valor que puede adjuntar a los recursos de AWS, incluidos los usuarios de IAM. Las etiquetas facilitan la gestión, la búsqueda y el filtrado de recursos.

También podemos emitir el siguiente comando para enumerar cualquier grupo del que podamos ser miembros. Un grupo de usuarios es una colección de usuarios de IAM, que permite a varios usuarios heredar los permisos que se le atribuyen. Descubrimos que el usuario de IAM no es miembro de ningún grupo.

```csharp
aws iam list-groups-for-user --user-name dev01
```

![image.png](image%206.png)

## **Listar políticas de usuario adjuntas (attached policies)**

En su lugar, podemos intentar enumerar las políticas de usuario adjuntas. 

```csharp
aws iam list-attached-user-policies --user-name dev01
```

![image.png](image%207.png)

Parece que tenemos permisos para enumerarlos, y esto revela que la política administrada de Amazon `AmazonGuardDutyReadOnlyAccess` y una política administrada por el cliente denominada `dev01` se han adjuntado a nuestro usuario de IAM. GuardDuty es un servicio de detección de amenazas que monitorea continuamente comportamientos maliciosos o no autorizados para ayudar a proteger sus cuentas y cargas de trabajo de AWS. El `PolicyArn` se refiere al nombre del 
recurso de Amazon, que es una combinación del ID de la cuenta de AWS, el tipo de recurso y el nombre del recurso. El ARN es una referencia global única al objeto. 

## Listar politicas en linea

A continuación, verifiquemos las políticas en línea. Una política en línea en AWS IAM es una política que está integrada directamente en un único usuario, grupo o rol de IAM y no se administra ni se puede reutilizar por separado. 

```csharp
aws iam list-user-policies --user-name dev01
```

![image.png](image%208.png)

Esto revela una interesante política en línea llamada `S3_Access` . ¡Veamos, examinemos las políticas y veamos qué permisos tenemos! 

## Profundizando en las politicas

Primero enumeramos las versiones del `AmazonGuardDutyReadOnlyAccess` política. Las políticas administradas por Amazon y por el cliente pueden tener varias versiones, lo que le permite conservar, revisar y revertir a versiones de políticas anteriores. Las políticas en línea no admiten el control de versiones. 

```csharp
aws iam list-policy-versions --policy-arn arn:aws:iam::aws:policy/AmazonGuardDutyReadOnlyAccess
```

![image.png](image%209.png)

La última versión es la v4. Al ejecutar el siguiente comando, vemos que todas las acciones `Describe`, `Get` y `List` están permitidas. 

```csharp
aws iam get-policy-version --policy-arn arn:aws:iam::aws:policy/AmazonGuardDutyReadOnlyAccess --version-id v4
```

![image.png](image%2010.png)

![image.png](image%2011.png)

`Describe` nos permite recuperar detalles sobre los recursos de GuardDuty. Se permitirá cualquier cosa que comience con "Describie" en las operaciones de la API de GuardDuty, como describir detectores, hallazgos o configuraciones. `Get` nos permite obtener hallazgos, conjuntos de información sobre amenazas o filtrar detalles, mientras que `List` permite enumerar detectores, miembros o hallazgos. Generalmente se utiliza para obtener una lista de recursos pero no su información detallada.

A continuación, enumeramos las versiones de la política administrada por el cliente denominada dev01 y ver que la última versión es v7.

```csharp
aws iam list-policy-versions --policy-arn arn:aws:iam::794929857501:policy/dev01 --profile lab
```

![image.png](image%2012.png)

Obtengamos también el documento JSON para esta política. Esto muestra varias acciones permitidas que se aplican a un usuario, rol y políticas en el la seccion Resources.

```csharp
aws iam get-policy-version --policy-arn arn:aws:iam::794929857501:policy/dev01 --version-id v7 --profile lab
```

![image.png](image%2013.png)

Podemos trazar las acciones permitidas según se aplican a los diversos recursos contenidos en el Resource sección del documento JSON:

1. IAM User (`arn:aws:iam::794929857501:user/dev01`):
- `iam:GetUser`
- `iam:ListAttachedUserPolicies`
- `iam:ListUserPolicies`
- `iam:GetUserPolicy`
- `iam:ListGroupsForUser`
2. IAM Role (`arn:aws:iam::794929857501:role/BackendDev`):
- `iam:GetRole`
- `iam:GetRolePolicy`
- `iam:ListAttachedRolePolicies`
3. Customer Managed Role Policy (`arn:aws:iam::794929857501:policy/BackendDevPolicy`):
- `iam:GetPolicy`
- `iam:GetPolicyVersion`
- `iam:ListPolicyVersions`
4. Customer Managed Policy (`arn:aws:iam::794929857501:policy/dev01`):
- `iam:GetPolicy`
- `iam:GetPolicyVersion`
- `iam:ListPolicyVersions`
5. Amazon Managed Policy (`arn:aws:iam::aws:policy/AmazonGuardDutyReadOnlyAccess`):
- `iam:GetPolicy`
- `iam:GetPolicyVersion`
- `iam:ListPolicyVersions`

Esta póliza administrada por el cliente adjunta al usuario `dev01`IAM otorgandole permisos para obtener detalles sobre sí mismo, cualquier grupo del que sea miembro y su política adjunta. También les otorga permiso para obtener detalles sobre la función BackendDev, enumerar las políticas adjuntas a la función y obtener información sobre una política denominada BackendDevPolicy que asumimos está adscrito al rol. También nos permite obtener información sobre una política 
administrada por Amazon.

Confirmemos nuestra suposición sobre BackendDev y BackendDevPolicy.

```csharp
aws iam list-attached-role-policies --role-name BackendDev --profile lab
```

![image.png](image%2014.png)

Sigamos enumerando los BackEnd rol y BackendDevPolicy política (ARN enumerados a continuación).

```csharp
arn:aws:iam::794929857501:role/BackendDev
arn:aws:iam::794929857501:policy/BackendDevPolicy
```

Podemos usar el get-role Iam acción para obtener información sobre el rol. Esto revela que el propósito del rol es permitir que los desarrolladores lo asuman. Asumir un papel puede considerarse un poco como el sudo comando en Linux, ya que proporciona credenciales temporales que nos permiten actuar en el contexto de privilegio del rol. Actualmente solo el usuario de IAM dev01 se le permite asumir el rol.

```csharp
aws iam get-role --role-name BackendDev --profile lab
```

![image.png](image%2015.png)

Continuemos nuestra investigación y enumeremos la política. La acción `get-policy` revela que la versión actual de la política es v1.

```csharp
aws iam get-policy --policy-arn arn:aws:iam::794929857501:policy/BackendDevPolicy --profile l
```

![image.png](image%2016.png)

Podemos usar `get-policy-version` para recuperar el documento de política JSON asociado con esta versión (actualmente la única versión). Si asumimos el rol, podremos recuperar información sobre todas las instancias EC2 en la cuenta de AWS y enumerar todos los secretos almacenados actualmente en SecretsManager. También podremos obtener información sobre (describir) y recuperar el valor secreto de prod/Customers !

```csharp
aws iam get-policy-version --policy-arn arn:aws:iam::794929857501:policy/BackendDevPolicy --version-id v1 --profile lab
```

![image.png](image%2017.png)

Finalmente, examinemos la política de usuario en línea con el get-user-policy acción. Esto muestra que podemos enumerar y recuperar cualquier contenido en el depósito S3 potencialmente confidencial llamado hl-dev-artifacts.

```csharp
aws iam get-user-policy --user-name dev01 --policy-name S3_Access --profile lab
```

![image.png](image%2018.png)

Tracemos visualmente a qué podría acceder alguien que use esta cuenta potencialmente comprometida.

![image.png](image%2019.png)

```csharp
aws s3 ls s3://hl-dev-artifacts --profile lab
```

![image.png](image%2020.png)

![image.png](image%2021.png)

## Extra Mile 

Asumir el rol y acceder al secreto de Secrets Manager.

[Como asumir un rol?](https://cloud.hacktricks.xyz/pentesting-cloud/aws-security/aws-privilege-escalation/aws-sts-privesc#sts-assumerole)

![image.png](image%2022.png)

![image.png](image%2023.png)

[Enumerar Secrets Manager](https://cloud.hacktricks.xyz/pentesting-cloud/aws-security/aws-services/aws-secrets-manager-enum#enumeration)

![image.png](image%2024.png)

![image.png](image%2025.png)

---

## References

- [Laboratorio en PwnedLabs](https://pwnedlabs.io/labs/intro-to-aws-iam-enumeration)
- [Enumeracion de IAM Hacktricks](https://cloud.hacktricks.xyz/pentesting-cloud/aws-security/aws-services/aws-iam-enum#enumeration)
- [STS: Asumir un Rol](https://cloud.hacktricks.xyz/pentesting-cloud/aws-security/aws-privilege-escalation/aws-sts-privesc#sts-assumerole)
- [Enumeracion de Secrets Manaer](https://cloud.hacktricks.xyz/pentesting-cloud/aws-security/aws-services/aws-secrets-manager-enum#enumeration)













