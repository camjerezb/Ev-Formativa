# Guía Práctica de Preparación: Gestión Avanzada de Estado y Recursos en Terraform

Esta actividad guiada está diseñada específicamente para prepararte para la **Evaluación Parcial 3** de la asignatura **Infraestructura como Código II (AUY1105)**. El objetivo es que domines la manipulación segura del archivo de estado de Terraform (`terraform.tfstate`) y resuelvas problemas comunes de desincronización y refactorización de infraestructura.

---

## 🎯 Objetivos de Aprendizaje (Indicadores de Logro)
* **IL 4.1:** Realizar operaciones de manipulación y refactorización en archivos de estado de Terraform, manteniendo la integridad de la infraestructura y evitando la pérdida de datos o configuraciones.
* **IL 4.2:** Aplicar técnicas de optimización y legibilidad en las configuraciones de Terraform.
* **IL 4.3:** Demostrar competencia en el uso de comandos avanzados de Terraform CLI (`terraform state`, `import`, `taint` / `untaint`, `refresh`).

---

## 🛠️ Requisitos Previos
1. Terraform CLI instalado en el sistema.
2. AWS CLI configurado con tus credenciales de laboratorio (AWS Academy o cuenta educativa).
3. Conectividad a Internet.
4. Editor de código (VS Code recomendado).

---

## 🏗️ Fase 0: Configuración e Infraestructura Inicial

Crea un directorio de trabajo en tu máquina e inicializa los siguientes archivos de configuración de Terraform.

### 1. Código Base: `main.tf`
Crea el archivo `main.tf` con la siguiente definición de infraestructura (VPC, Subnet, Security Group y una Instancia EC2):

```hcl
provider "aws" {
  region = "us-east-1"
}

# 1. Definición de la VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name = "VPC-Actividad-Guiada"
  }
}

# 2. Definición de la Subnet
resource "aws_subnet" "main" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true

  tags = {
    Name = "Subnet-Actividad-Guiada"
  }
}

# 3. Definición del Grupo de Seguridad
resource "aws_security_group" "web" {
  name        = "sg_web_actividad"
  description = "Grupo de seguridad para actividad guiada"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "SG-Actividad-Guiada"
  }
}

# 4. Definición de la Instancia EC2
resource "aws_instance" "web" {
  ami                    = "ami-0c7217cdde317cfec" # Amazon Linux 2 AMI en us-east-1
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.main.id
  vpc_security_group_ids = [aws_security_group.web.id]

  tags = {
    Name = "EC2-Actividad-Guiada"
  }
}
```

### 2. Despliegue de la infraestructura inicial
Abre tu terminal en la carpeta del proyecto y ejecuta:

```bash
# Inicializar el proveedor y módulos
terraform init

# Validar que la sintaxis sea correcta
terraform validate

# Aplicar los cambios para crear la infraestructura real
terraform apply -auto-approve
```

> [!IMPORTANT]
> Una vez finalizado el despliegue, verifica que se haya creado el archivo `terraform.tfstate` en tu directorio local. **¡No borres este directorio!** Lo usaremos como base.

---

## ⚡ Escenario 1: Recuperación del Estado de Terraform (Importar Recursos)

**Situación:** Por un error de sistema o descuido humano, el archivo de estado de tu infraestructura se ha eliminado por completo, pero los recursos siguen corriendo y activos en AWS. Debes reconstruir el estado sin destruir ni recrear la infraestructura.

### Paso 1.1: Simular la pérdida del estado
Elimina manualmente los archivos de estado de tu carpeta local:
* Borra `terraform.tfstate`
* Borra `terraform.tfstate.backup` (si existe)

### Paso 1.2: Identificar el impacto
Ejecuta un plan para ver qué cree Terraform que debe hacer:
```bash
terraform plan
```
📝 *Pregunta para reflexionar:* ¿Por qué Terraform te indica que creará 4 recursos nuevos si estos ya existen en tu cuenta de AWS? 
*(Respuesta: Al no tener el archivo `terraform.tfstate`, Terraform no tiene registro de mapeo entre tu código y la nube).*

### Paso 1.3: Obtener los IDs reales desde AWS
Ve a tu Consola de AWS (o usa la AWS CLI) y anota los IDs físicos de los siguientes recursos creados en la Fase 0:
* **VPC ID** (ej: `vpc-0123456789abcdef0`)
* **Subnet ID** (ej: `subnet-0123456789abcdef0`)
* **Security Group ID** (ej: `sg-0123456789abcdef0`)
* **EC2 Instance ID** (ej: `i-0123456789abcdef0`)

### Paso 1.4: Reconstruir el estado usando `terraform import`
Debes asociar cada recurso del código a su contraparte física en la nube mediante el comando `terraform import <direccion_recurso_codigo> <id_recurso_real>`. 

Ejecuta los siguientes comandos uno por uno reemplazando los IDs de ejemplo por tus IDs reales:

```bash
# 1. Importar la VPC
terraform import aws_vpc.main vpc-0123456789abcdef0

# 2. Importar la Subnet
terraform import aws_subnet.main subnet-0123456789abcdef0

# 3. Importar el Security Group
terraform import aws_security_group.web sg-0123456789abcdef0

# 4. Importar la Instancia EC2
terraform import aws_instance.web i-0123456789abcdef0
```

### Paso 1.5: Verificar el estado recuperado
Una vez completadas las importaciones, valida que los recursos estén correctamente registrados en el estado local:

```bash
# Listar los recursos registrados en el estado
terraform state list

# Inspeccionar los detalles de la instancia EC2 importada en el estado
terraform state show aws_instance.web
```

### Paso 1.6: Validación final
Ejecuta un plan final para asegurarte de que tu código y tu estado estén 100% sincronizados con AWS:
```bash
terraform plan
```
> [!TIP]
> Si la sincronización fue exitosa, el comando mostrará: **"No changes. Your infrastructure matches the configuration."** Si detecta cambios, ajusta los atributos en tu archivo `main.tf` para que coincidan exactamente con la realidad importada.

---

## ⚡ Escenario 2: Sincronización, Forzar Recreación (`taint`) y Revertir (`untaint`)

**Situación:** Se han realizado modificaciones manuales a la infraestructura desde la consola web de AWS (desviación o *configuration drift*). Debes sincronizar los cambios de estado. Además, necesitas forzar la recreación de un recurso específico que está fallando o requiere ser reconstruido desde cero.

### Paso 2.1: Provocar desviación manual (*Drift*)
1. Ve a la consola de AWS, busca el Security Group creado (`sg_web_actividad`).
2. Edita las reglas de entrada (*Inbound Rules*).
3. **Agrega manualmente** una nueva regla (por ejemplo, habilitar puerto `22` (SSH) desde `0.0.0.0/0`).
4. Guarda los cambios en la consola.

### Paso 2.2: Detectar la desviación
Ejecuta un plan para ver cómo Terraform detecta la diferencia entre tu código y la realidad en AWS:
```bash
terraform plan
```
Observarás que Terraform detecta la regla adicional y propone modificar el Security Group para volver al estado descrito en el código.

### Paso 2.3: Sincronizar el estado mediante `terraform refresh`
Si quieres que el archivo de estado de Terraform se actualice con los valores de la nube sin aplicar cambios del código, puedes usar:
```bash
terraform refresh
```
*(Nota: En versiones modernas de Terraform, esto se hace implícitamente en cada plan/apply o usando `terraform plan -refresh-only`).* 
Revisa el archivo de estado o ejecuta `terraform state show aws_security_group.web` para validar que ahora el estado conoce el puerto 22.

### Paso 2.4: Forzar la recreación de un recurso usando `taint`
Imagina que la instancia EC2 ha sufrido corrupción de software y necesitas que en el próximo despliegue sea completamente destruida y vuelta a crear (recreación limpia).

```bash
# Marcar la instancia como "contaminada" (taint)
terraform taint aws_instance.web
```

Ejecuta un plan para ver el efecto de esta marcación:
```bash
terraform plan
```
📝 *Observa la salida:* Verás que Terraform marca la instancia con el mensaje: **"is tainted, so must be replaced"** (será destruida y recreada).

### Paso 2.5: Revertir la marcación usando `untaint`
Si te arrepientes o decides que no es necesario recrear el recurso físicamente, puedes quitar la marca de contaminación:

```bash
# Desmarcar la instancia
terraform untaint aws_instance.web
```

Ejecuta un plan para comprobar que la instancia ya no será destruida:
```bash
terraform plan
```
*(Debería mostrar que no hay cambios pendientes para la instancia EC2).*

### Paso 2.6: Aplicar recreación definitiva
Para practicar el flujo completo de `taint`:
1. Vuelve a marcar la instancia: `terraform taint aws_instance.web`
2. Aplica los cambios: `terraform apply -auto-approve`
3. Comprueba en la consola de AWS que la instancia anterior se está terminando y una nueva se está inicializando (con un nuevo ID).
4. Vuelve a importar o sincronizar si es necesario, aunque al aplicar el plan el estado se actualiza automáticamente con el nuevo ID de la instancia.

---

## ⚡ Escenario 3: Desasociar Recursos del Estado (`state rm`)

**Situación:** El equipo de seguridad ha decidido que el Security Group ahora se gestionará manualmente o mediante otra herramienta externa. Debes remover este recurso del control de Terraform, asegurándote de que no sea destruido físicamente en AWS cuando elimines el código.

### Paso 3.1: Identificar el recurso en el estado
Lista todos tus recursos para confirmar la ruta del Security Group:
```bash
terraform state list
```
*(Deberías ver `aws_security_group.web` en la lista).*

### Paso 3.2: Remover el recurso del estado
Ejecuta el comando para remover el recurso del archivo de estado local sin tocar la infraestructura en AWS:

```bash
terraform state rm aws_security_group.web
```
📝 *Verificación:* Ejecuta `terraform state list` para confirmar que ya no aparece en el listado del estado de Terraform.

### Paso 3.3: Ajustar el código
Si ejecutas `terraform plan` en este momento, ¿qué pasará?
Terraform detectará que en tu código `main.tf` tienes definido el bloque `resource "aws_security_group" "web"`, pero no está en su estado, por lo que **tratará de crearlo de nuevo** (generando un error por conflicto de nombre si ya existe).

Para evitar esto, debes eliminar o comentar el bloque de código correspondiente al Security Group en tu archivo `main.tf`:

```hcl
# COMENTAR O ELIMINAR ESTE BLOQUE EN EL ARCHIVO main.tf:
# resource "aws_security_group" "web" {
#   ...
# }
```

> [!WARNING]
> La instancia EC2 en el código hace referencia a `aws_security_group.web.id`. Si comentas el bloque del SG, la referencia en la instancia EC2 fallará. 
> Debes modificar la configuración de la instancia EC2 en `main.tf` para que use el ID del Security Group como un string directo o remover la asociación temporalmente:
> 
> ```hcl
> resource "aws_instance" "web" {
>   ...
>   # Reemplaza la referencia dinámica por el ID de string del Security Group real
>   vpc_security_group_ids = ["sg-XXXXXXXXXXXXXX"] 
> }
> ```

### Paso 3.4: Validar la persistencia física del recurso
1. Ejecuta `terraform plan` para validar que todo esté sincronizado.
2. Ingresa a la consola de AWS.
3. Verifica que el Security Group (`sg_web_actividad`) **siga existiendo físicamente**. 
*(Esto demuestra que `terraform state rm` remueve el control lógico del recurso en el estado de Terraform, pero no destruye el recurso real en la nube).*

---

## 🧹 Limpieza de Recursos (Destrucción)
Una vez que hayas terminado tu práctica, destruye los recursos para evitar cobros innecesarios en tu cuenta de AWS:

```bash
terraform destroy -auto-approve
```

*(Nota: Dado que removiste el Security Group del estado, este no se destruirá con el comando anterior. Tendrás que eliminar el Security Group manualmente desde la consola de AWS una vez terminada la actividad).*

---

## ✍️ Formato de Entrega Recomendado para la Prueba
Para tu evaluación de la próxima semana, se te solicitará un informe paso a paso. Acostúmbrate a tomar capturas de:
1. El comando ejecutado en la terminal.
2. La salida que entrega la consola de Terraform (especialmente los mensajes de éxito, warnings o planes de adición/modificación/destrucción).
3. La consola web de AWS donde se evidencien los IDs, cambios manuales o persistencia de los recursos.

¡Éxito en tu preparación! Practicar estos comandos te dará la seguridad y velocidad necesarias para rendir una excelente evaluación.
