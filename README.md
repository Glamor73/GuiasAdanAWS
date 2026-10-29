# GuiasAdanAWS
# Taller 1. Preparar la cuenta y la red de la asignatura (AWS)

## Objetivo

Organizar tu cuenta de AWS (Amazon Web Services) para la asignatura **Infraestructura y Cloud**: etiquetar los recursos de la asignatura, crear un usuario IAM (Identity and Access Management, gestión de identidades y accesos) con permisos limitados y preparar la red donde estará la página del taller de publicación. Usarás tu cuenta con el plan gratuito de AWS y sus créditos iniciales.

**Tiempo estimado: 50 minutos.**

## Equivalencias OCI → AWS

| OCI (guía original) | AWS (este taller) |
| --- | --- |
| Compartimento `Infraestructura_y_Cloud` | No existe; se usa la **etiqueta** `Asignatura = Infraestructura_y_Cloud` y un **Resource Group** |
| Grupo + Política | **User group** + políticas administradas de IAM |
| Usuario IAM | **IAM user** |
| VCN | **VPC** (`10.0.0.0/16`) |
| Internet Gateway + Route Table | **Internet gateway** + **Route table** |
| Subred regional | **Subnet** (en AWS cada subred vive en **una** zona de disponibilidad) |
| Security List | **Security Group** (stateful, se asocia a la máquina) |

## Antes de empezar

- Usa una sola región en los dos talleres (recomendado: **US East (N. Virginia) `us-east-1`**, suele ser la más barata y completa).
- Con el plan gratuito, el crédito se consume por uso. Lo que crearás en este taller (VPC, subred, gateway, security group) **no tiene costo**. Evita crear **NAT Gateway** o **Elastic IP** sin usar: sí cobran.
- Nunca guardes contraseñas ni llaves de acceso en `datos-taller.txt`.

---

## Fase 1. Organizar la cuenta y preparar el acceso

### Paso 1.1. Ingresar y registrar los datos de la cuenta

1. Entra a `console.aws.amazon.com` y elige **Root user** con el correo de tu cuenta de AWS. Mantén esta sesión abierta durante el taller.
2. En la barra superior (esquina derecha) elige la región `us-east-1` (N. Virginia). Úsala en todos los pasos.
3. Abre **Billing and Cost Management > Credits** y revisa el saldo de créditos y su fecha de vencimiento. Revisa también **Free Tier**.
4. Crea en Visual Studio Code el archivo `datos-taller.txt` y guarda: ID de cuenta (12 dígitos), región y saldo de créditos.
5. Activa el acceso del usuario IAM a la facturación: ve al nombre de tu cuenta (arriba a la derecha) > **Account** > sección **IAM user and role access to Billing information** > **Edit** > **Activate IAM access** > **Update**. Solo el usuario root puede hacerlo.

### Paso 1.2. Organizar los recursos de la asignatura (etiqueta y Resource Group)

AWS no tiene compartimentos. Agruparás los recursos con una etiqueta.

1. En el buscador de la consola escribe **Resource Groups & Tag Editor** y ábrelo.
2. Selecciona **Create resource group** > **Tag based**.
3. Completa:

| Campo | Valor |
| --- | --- |
| Grouping criteria: Resource types | All supported resource types |
| Tag key | `Asignatura` |
| Tag value | `Infraestructura_y_Cloud` |
| Group name | `Infraestructura_y_Cloud` |
| Description | `Recursos de la asignatura Infraestructura y Cloud` |

4. Pulsa **Create group**.
5. **Regla de oro:** a todo recurso que crees (VPC, gateway, subred, security group, y en el taller 2 la instancia) agrégale la etiqueta `Asignatura = Infraestructura_y_Cloud`. Guarda esto en `datos-taller.txt`.

### Paso 1.3. Crear el grupo de permisos

1. Abre **IAM > User groups > Create group**.
2. En **User group name** escribe `GrupoInfraestructura`.
3. En **Attach permissions policies** busca y marca:
   - `AmazonEC2FullAccess` (servidores, discos y también la red VPC)
   - `AmazonVPCFullAccess`
   - `AWSCloudShellFullAccess`
4. Pulsa **Create user group** y guarda el nombre en `datos-taller.txt`.

### Paso 1.4. Crear la política de costos

Permitirá al grupo consultar costos y crear alertas de presupuesto.

1. Abre **IAM > Policies > Create policy**, cambia a la pestaña **JSON** y pega:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "budgets:ViewBudget",
        "budgets:ModifyBudget",
        "ce:Get*",
        "ce:Describe*",
        "ce:List*",
        "billing:Get*",
        "billing:List*",
        "freetier:Get*",
        "account:GetAccountInformation"
      ],
      "Resource": "*"
    }
  ]
}
```

2. Pulsa **Next**, escribe `PermisosInfraestructura` en **Policy name** y `Permisos para publicar y controlar el gasto del taller` en **Description**. Pulsa **Create policy**.
3. Vuelve a **IAM > User groups > GrupoInfraestructura > Permissions > Add permissions > Attach policies**, marca `PermisosInfraestructura` y pulsa **Add permissions**.
4. Si la consola rechaza alguna acción del JSON, quita esa línea y vuelve a guardar. Registra `PermisosInfraestructura` en `datos-taller.txt`.

### Paso 1.5. Crear el usuario de práctica

1. Abre **IAM > Users > Create user**.
2. En **User name** escribe `estudiante-web`.
3. Marca **Provide user access to the AWS Management Console** > **I want to create an IAM user**.
4. Elige **Custom password**, escribe una contraseña personal y deja marcado **Users must create a new password at next sign-in**. Pulsa **Next**.
5. En **Set permissions** elige **Add user to group** y marca solo `GrupoInfraestructura`. Pulsa **Next**.
6. En **Tags** agrega `Asignatura = Infraestructura_y_Cloud`. Pulsa **Create user**.
7. En la pantalla final, copia la **Console sign-in URL** (tiene tu ID de cuenta) y guárdala en `datos-taller.txt`. No des permisos de administrador a este usuario.

### Paso 1.6. Activar el nuevo usuario

AWS no envía correo de bienvenida en este flujo: la activación ocurre en el primer inicio de sesión.

1. Abre una **ventana privada** del navegador y entra a la **Console sign-in URL** del paso anterior.
2. Inicia sesión con `estudiante-web` y la contraseña temporal; la consola te pedirá crear una contraseña definitiva.
3. (Recomendado) Activa MFA: **IAM > Users > estudiante-web > Security credentials > Assign MFA device**.

### Paso 1.7. Comprobar el acceso y guardar el resultado

1. Con `estudiante-web` en la ventana privada, elige la región `us-east-1`.
2. Abre **EC2 > Instances**, pulsa **Launch instances** y **cancela** sin crear nada.
3. Abre **VPC > Your VPCs** para confirmar que tienes acceso a la red.
4. Si ves "You are not authorized", revisa en la sesión root que el grupo tenga las tres políticas del paso 1.3 y la del 1.4; cierra sesión y vuelve a entrar.
5. Guarda `datos-taller.txt` y una captura de la lista de instancias con la región visible. Conserva el archivo para el taller 2.

---

## Fase 2. Crear la red de la asignatura

Continúa con la sesión de `estudiante-web` y la región `us-east-1`. Recuerda etiquetar **cada recurso** con `Asignatura = Infraestructura_y_Cloud`.

### Paso 2.1. Crear la red virtual (VPC)

1. Abre **VPC > Your VPCs > Create VPC**.
2. Elige **VPC only** y completa:

| Campo | Valor |
| --- | --- |
| Name tag | `red-infraestructura` |
| IPv4 CIDR manual input | `10.0.0.0/16` |
| IPv6 CIDR block | No IPv6 CIDR block |
| Tenancy | Default |
| Tag adicional | `Asignatura = Infraestructura_y_Cloud` |

3. Pulsa **Create VPC**.
4. Selecciona la VPC > **Actions > Edit VPC settings** > marca **Enable DNS hostnames** > **Save**.
5. Guarda `red-infraestructura` en `datos-taller.txt`.

### Paso 2.2. Crear la salida a Internet y su ruta

1. Abre **VPC > Internet gateways > Create internet gateway**. Name: `salida-internet`, agrega la etiqueta de la asignatura y pulsa **Create**.
2. Con el gateway seleccionado: **Actions > Attach to VPC** > elige `red-infraestructura` > **Attach internet gateway**.
3. Abre **VPC > Route tables > Create route table**: Name `rutas-web`, VPC `red-infraestructura`, etiqueta de la asignatura > **Create route table**.
4. En `rutas-web` abre **Routes > Edit routes > Add route**:

| Campo | Valor |
| --- | --- |
| Destination | `0.0.0.0/0` |
| Target | **Internet Gateway** > `salida-internet` |

5. Pulsa **Save changes**. El destino `0.0.0.0/0` representa todas las direcciones externas.

### Paso 2.3. Crear la subred pública

1. Abre **VPC > Subnets > Create subnet** y elige la VPC `red-infraestructura`.
2. Completa:

| Campo | Valor |
| --- | --- |
| Subnet name | `subred-web` |
| Availability Zone | `us-east-1a` |
| IPv4 subnet CIDR block | `10.0.0.0/24` |
| Tag adicional | `Asignatura = Infraestructura_y_Cloud` |

3. Pulsa **Create subnet**.
4. Selecciona `subred-web` > **Actions > Edit subnet settings** > marca **Enable auto-assign public IPv4 address** > **Save**. Esto equivale a "Public Subnet" en OCI.
5. Pestaña **Route table > Edit route table association**, elige `rutas-web` y pulsa **Save**.
6. Verifica en los detalles: CIDR `10.0.0.0/24`, auto-assign público activado y tabla `rutas-web` con la ruta `0.0.0.0/0` al gateway.
7. Guarda `subred-web` en `datos-taller.txt`.

> En AWS no creas DHCP ni DNS label: la VPC ya trae un DHCP options set por defecto.

### Paso 2.4. Permitir la conexión y el acceso a la página (Security Group)

El puerto **22** permite SSH (administración) y el **80** permite HTTP (la página).

1. Abre **VPC > Security groups > Create security group**.
2. Completa: Name `sg-web`, Description `Acceso SSH y HTTP del taller`, VPC `red-infraestructura`.
3. En **Inbound rules > Add rule** crea:

| Type | Protocol | Port range | Source |
| --- | --- | --- | --- |
| SSH | TCP | 22 | **My IP** (más seguro) o `0.0.0.0/0` |
| HTTP | TCP | 80 | `0.0.0.0/0` |

4. En **Outbound rules** deja la regla por defecto: All traffic, `0.0.0.0/0`.
5. Agrega la etiqueta de la asignatura y pulsa **Create security group**.
6. A diferencia de OCI, el security group se asocia a la **instancia** al crearla (taller 2); selecciona `sg-web` en ese momento. Los security groups son stateful: las respuestas se permiten automáticamente.
7. Verifica en **Subnets > subred-web** que use la tabla `rutas-web`. Guarda `sg-web` en `datos-taller.txt`.

---

## Resumen para `datos-taller.txt`

```text
Cuenta (ID): ...
Región: us-east-1
Saldo de créditos: ...
Etiqueta: Asignatura = Infraestructura_y_Cloud
Grupo: GrupoInfraestructura
Política: PermisosInfraestructura
Usuario: estudiante-web
Sign-in URL: ...
VPC: red-infraestructura (10.0.0.0/16)
Gateway: salida-internet
Tabla de rutas: rutas-web
Subred: subred-web (10.0.0.0/24, us-east-1a)
Security group: sg-web
```
