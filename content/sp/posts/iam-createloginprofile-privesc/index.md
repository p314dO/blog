---
title: IAM CreateLoginProfile PrivEsc - Cybr
date: 2024-11-27
description: Privilege escalation through create login profile.
isStarred: true
draft: true
---

![](1.jpeg)
Explote iam:CreateLoginProfile para obtener acceso a un depósito de Amazon S3 que contiene datos confidenciales a los que su usuario de IAM. La bandera es el número de SSN de "Holly Duncan" del archivo "ssn.csv".

## Escenario

El objetivo de este laboratorio es explotar la vulnerabilidad `iam:CreateLoginProfile`  para otorgarse permisos a los que no debería tener acceso. Ha completado con éxito esta práctica de laboratorio una vez que haya accedido y descargado archivos confidenciales que contienen 
información de identificación personal del cliente en Amazon S3 y haya enviado el número de SSN de "Holly Duncan".

