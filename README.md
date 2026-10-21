# Aprovisionamiento Automatizado de Servidores Web en AWS con Terraform

**Entorno Técnico:**
- Proveedor Cloud: Amazon Web Services (AWS)
- Infraestructura por Código (IaC): Terraform (HashiCorp)
- SO: Ubuntu 24.04 LTS
- Servicio Web: Apache2

**Objetivo Empresarial:**
Desplegar infraestructura web escalable mediante código. Eliminar la configuración manual, asegurar la reproducibilidad de los entornos y gestionar el ciclo de vida de los recursos en la nube bajo el paradigma de Infraestructura como Código (IaC).

## Guía de Despliegue (Ubuntu Linux)

**1. Instalación de dependencias y binarios oficiales:**
Preparación del entorno base mediante el gestor de paquetes de la distribución. Descarga e instalación de la interfaz de línea de comandos de AWS y el motor de Terraform.

```bash
# Herramientas base
sudo apt update && sudo apt install -y curl unzip gpg lsb-release

# Instalación de AWS CLI v2
curl "[https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip](https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip)" -o "awscliv2.zip"
unzip -q awscliv2.zip
sudo ./aws/install
rm -rf aws awscliv2.zip

# Adición del repositorio de HashiCorp e instalación de Terraform
wget -O- [https://apt.releases.hashicorp.com/gpg](https://apt.releases.hashicorp.com/gpg) | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] [https://apt.releases.hashicorp.com](https://apt.releases.hashicorp.com) $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform
```

Verificación de versiones instaladas:

```Bash
aws --version && terraform -version
```

**2. Inyección de credenciales de acceso:**
Configuración del perfil local para la autenticación programática contra la API de AWS.

```Bash
mkdir -p ~/.aws
cat <<EOF> ~/.aws/credentials
[default]
aws_access_key_id=TU_ACCESS_KEY
aws_secret_access_key=TU_SECRET_KEY
aws_session_token=TU_SESSION_TOKEN
EOF
```

Verifica la trazabilidad. La terminal devolverá el ARN de la cuenta operativa:

```Bash
aws sts get-caller-identity
```

**3. Aislamiento del estado y control de versiones:**
Generación del directorio de trabajo y protección de estados internos frente a fugas en repositorios públicos.

```Bash
mkdir -p ~/servidor-web-iac && cd ~/servidor-web-iac
cat <<EOF> .gitignore
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars
EOF
```

**4. Definición de la infraestructura (HCL):**
Creación del manifiesto declarativo. Configura la región, localiza la AMI de Ubuntu dinámica, establece reglas de cortafuegos perimetral (puerto 80) e inyecta el script de aprovisionamiento desatendido para Apache.

```terraform
cat << 'EOF' > main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_security_group" "web_sg" {
  name        = "web-server-sg"
  description = "Permitir trafico HTTP entrante"

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
}

resource "aws_instance" "servidor_web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  user_data = <<-USERDATA
              #!/bin/bash
              apt-get update -y
              apt-get install -y apache2
              echo "<h1>Servidor aprovisionado mediante Terraform en AWS</h1>" > /var/www/html/index.html
              systemctl enable apache2
              systemctl start apache2
              USERDATA

  tags = {
    Name = "Web-Node-01"
  }
}

output "servidor_ip" {
  value = aws_instance.servidor_web.public_ip
}
EOF
```

**5. Construcción y validación:**
Inicialización de módulos y despliegue físico de la infraestructura.

```bash
terraform init
terraform apply -auto-approve
```

La consola devolverá la IP pública asignada (servidor_ip = "XX.XX.XX.XX"). Tras 60 segundos de espera para la ejecución del script interno, valida el acceso al servidor web:

```Bash
curl http://PEGA_AQUI_LA_IP
```

**6. Destrucción de recursos (Cost Optimization):**
Eliminación automática de la máquina virtual y reglas de red para congelar la facturación.

```Bash
terraform destroy -auto-approve
```
