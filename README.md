# AWS Infrastructure

Deploying an API Rest over Amazon Web Services (AWS) requires the configuration of S3, ECR, Lambda, and API Gateway services. 
The following image presents the infrastructure design for the processing agent, one of the two components (agents) of the traffic accident risk level prediction system.

<img src="aws_infrastructure_prevention_system.png" width="800"/>

# Model deployment in ECR, S3, Lambda and API Gateway

## Índice

1. [Pasos Previos](#pasos-previos)
2. [Subir el Modelo a S3](#1-subir-el-modelo-a-s3)
3. [Subir la Imagen a Amazon ECR](#2-subir-la-imagen-a-amazon-ecr)
4. [Crear la Función Lambda](#3-crear-la-función-lambda)
5. [Crear la API en API Gateway](#4-crear-la-api-en-api-gateway)
6. [Probar la API](#5-probar-la-api)
7. [Actualizar la Función Lambda](#6-actualizar-la-función-lambda)

---

## 1.Pasos Previos

1. **Requisitos iniciales**:
    - Tener un bucket en S3 con el modelo `.joblib` disponible.
    - Crear un repositorio en ECR (ejemplo: `mlp-predictor-repo`).
    - Instalar AWS CLI en Ubuntu.

    Si no tienes AWS CLI instalado, ejecuta los siguientes comandos:

    #### En Linux:
    ```bash
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install
    aws --version
    ```

    #### En Windows:
    1. Descarga el instalador desde el siguiente enlace: [AWS CLI para Windows](https://awscli.amazonaws.com/AWSCLIV2.msi).
    2. Ejecuta el archivo `.msi` y sigue las instrucciones del asistente de instalación.
    3. Verifica la instalación abriendo una terminal de PowerShell y ejecutando:
        ```powershell
        aws --version
        ```

2. **Configurar la terminal**:
    - Crea un usuario en IAM y genera las credenciales de acceso (`access_key`).
    - Configura AWS CLI con el siguiente comando:

      ```bash
      aws configure
      ```

      Introduce los valores solicitados:
      - `AWS Access Key ID`: YOUR_ACCESS_KEY_ID
      - `AWS Secret Access Key`: YOUR_SECRET_ACCESS_KEY
      - `Default region name`: REGION
      - `Default output format`: json

    - Verifica la configuración con:

      ```bash
      aws sts get-caller-identity
      ```

      Ejemplo de salida:

      ```json
      {
            "UserId": "USER_ID",
            "Account": "ACCOUNT_ID",
            "Arn": "arn:aws:iam::ACCOUNT_ID:user/YOUR_IAM_NAME"
      }
      ```

    - Configura ECR para Docker:

      ```bash
      aws ecr get-login-password --region REGION | docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com
      ```

3. **Creación de usuarios y roles**:
    - **Usuario**:
      - Nombre: `YOUR_IAM_NAME` (ejemplo: `jperez`).
      - Políticas de permisos:
         - `AmazonEC2ContainerRegistryFullAccess`
         - `AWSLambda_FullAccess`
         - `AmazonAPIGatewayAdministrator`
         - `AmazonS3FullAccess`

    - **Rol**:
      - Nombre: `YOUR_ROLE_NAME` (ejemplo: `mi-rol-lambda`).
      - Caso de uso: selecciona **AWS service** y elige **Lambda**.
      - Políticas de permisos:
         - `AWSLambdaBasicExecutionRole`
         - `AWSLambda_FullAccess`
         - `AmazonS3ReadOnlyAccess`
         - Política personalizada (opcional):

            ```json
            {
                 "Version": "2012-10-17",
                 "Statement": [
                      {
                            "Sid": "VisualEditor0",
                            "Effect": "Allow",
                            "Action": [
                                 "ec2:CreateNetworkInterface",
                                 "ec2:DescribeNetworkInterfaces",
                                 "ec2:DescribeVpcs",
                                 "ec2:DeleteNetworkInterface",
                                 "ec2:DescribeSubnets",
                                 "ec2:DescribeSecurityGroups"
                            ],
                            "Resource": "*"
                      }
                 ]
            }
            ```

---

## **1. Subir el Modelo a S3**

1. En el servicio S3, selecciona la opción **Buckets** en el panel izquierdo.
2. Haz clic en **Crear bucket**.

    ![image.png](imgages/image.png)

    - Configura el bucket con un nombre único (este nombre será usado en el script de Lambda).
    - Deja las configuraciones por defecto.

3. Sube el archivo del modelo (`model.joblib`).

    ![image.png](images/image_1.png)

4. Edita el archivo `inference.py` para actualizar el nombre del bucket.

    ![image.png](images/image_2.png)

---

## **2. Subir la Imagen a Amazon ECR**

1. En el servicio ECR, selecciona **Repositorios** en el panel izquierdo y haz clic en **Crear repositorio**.

    ![image.png](images/image_3.png)

2. Asigna un nombre al repositorio y selecciona la opción **Mutable**.

    ![image.png](images/image_4.png)

3. Toma nota de la URL del repositorio, ya que será necesaria para los siguientes pasos.

    ![image.png](images/image_5.png)

### Preparar el repositorio

- Asegúrate de tener configurada la consola de AWS y estar en el directorio que contiene los archivos `Dockerfile` y `inference.py`.

### Construir la Imagen Docker

1. Ubicarse en el directorio:
   ```bash
    cd    Despliegue_lambda_apigateway_ecr/Manual Despliegue modelo en ECR, S3, Lambda y api gateway
    ```
3. Verifica que los archivos `Dockerfile` y `inference.py` estén en la misma carpeta.
4. Ejecuta el siguiente comando:

    ```bash
    docker build -t mlp-model .
    ```

### Etiquetar y Subir la Imagen a ECR

1. Etiqueta la imagen con la URI del repositorio de ECR:

    ```bash
    docker tag mlp-model:latest ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/NOMBRE_REPOSITORIO_ECR:latest
    ```

2. Sube la imagen al repositorio de ECR:

    ```bash
    docker push ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/NOMBRE_REPOSITORIO_ECR:latest
    ```

---

## **3. Crear la Función Lambda**

1. En el servicio Lambda, selecciona **Funciones** en el panel izquierdo y haz clic en **Crear función**.

    ![image.png](images/image_6.png)

2. Selecciona la opción **Imagen de contenedor**, asigna un nombre a la función y haz clic en **Examinar imágenes**.

    ![image.png](images/image_7.png)

3. Selecciona la imagen del repositorio creado previamente.

    ![image.png](images/image_9.png)

4. Configura la arquitectura como `x86_64` (por defecto) y asigna el rol creado con los permisos necesarios.

    ![image.png](images/image_11.png)

5. En la pestaña **Probar**, crea un evento JSON de prueba y haz clic en **Probar**.

    ```json
    {
         "body": "{\"Input\": [[0.5, 2.3, 1.2, 3.4, 0.9, 98, 12.4, 72, 120, -34.6, -58.4, 2, 0]]}",
         "headers": {
              "Content-Type": "application/json"
         },
         "httpMethod": "POST",
         "isBase64Encoded": false,
         "path": "/predict",
         "queryStringParameters": {},
         "requestContext": {
              "httpMethod": "POST",
              "requestId": "test-request-id"
         }
    }
    ```

    ![image.png](images/image_12.png)

6. Si obtienes un error, ajusta el tiempo de espera en la pestaña **Configuración**.

    ![image.png](images/image_15.png)

    - Incrementa el tiempo de espera a 20 segundos y guarda los cambios.

    ![image.png](images/image_16.png)

7. Vuelve a realizar la prueba. Una vez que funcione correctamente, reduce el tiempo de espera a 3 segundos.

    ![image.png](images/image_19.png)

---

## **4. Crear la API en API Gateway**

1. En el servicio API Gateway, selecciona **Crear API**.

    ![image.png](images/servicio_gateway.png)

2. Configura los parámetros:
    - Tipo de API: **Nueva API**.
    - Nombre de la API.
    - Tipo de punto de conexión: **Regional**.

    ![image.png](images/crear_api_gateway.png)

3. En el menú **Recurso**, selecciona **Crear recurso**.

    ![image.png](images/image_21.png)

4. Crea un método POST y configura los detalles:
    - Tipo de integración: **Función Lambda**.
    - Región y ARN de la función Lambda.

    ![Crear_Metodo_POST.PNG](images/crear_metodo_post.png)

5. Implementa la API para obtener la URL de invocación.

    ![Deploy_API_Gateway (1).PNG](images/deploy_api_gateway.png)

    Ejemplo de URL:

    ```plaintext
    https://{nombre-api}.execute-api.{region}.amazonaws.com/{stage-name}/{resource-path}
    ```

---

## **5. Probar la API**

1. Usa **Postman** para probar la API.
2. Configura el método como POST y pega la URL generada.

    ![image.png](images/image_22.png)

3. En el cuerpo de la solicitud, selecciona **raw** y pega el siguiente input:

    ```json
    {"Input": [[0.5, 2.3, 1.2, 3.4, 0.9, 98, 12.4, 72, 120, -34.6, -58.4, 2, 0]]}
    ```

    ![image.png](images/image_23.png)

4. Asegúrate de que el encabezado tenga `Content-Type: application/json`.

    ![image.png](images/image_24.png)

---

## **6. Actualizar la Función Lambda**

1. Realiza los cambios necesarios en el archivo `inference.py` y guarda los cambios.
2. Reconstruye la imagen Docker:

    ```bash
    docker build -t mlp-model .
    ```

3. Etiqueta y sube la nueva imagen a ECR:

    ```bash
    docker tag mlp-model:latest ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/NOMBRE_REPOSITORIO_ECR:latest
    docker push ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/NOMBRE_REPOSITORIO_ECR:latest
    ```

4. Actualiza la función Lambda con la nueva imagen:

    - Ve a la pestaña **Imagen** de la función Lambda.
    - Selecciona **Implementar nueva imagen**.

      ![image.png](images/image_26.png)

    - Guarda los cambios.

      ![image.png](images/image_27.png)

    La función se actualizará correctamente.

    ![image.png](images/image_28.png)


# Database deployment in RDS, Lambda, and API Gateway

## Índice

1. [Creación de la Base de Datos en Amazon RDS](#creación-de-la-base-de-datos-en-amazon-rds)
2. [Conexión de pgAdmin a la Instancia de RDS](#conexión-de-pgadmin-a-la-instancia-de-rds)
3. [Subir Tablas y Funciones a la Base de Datos en RDS](#subir-tablas-y-funciones-a-la-base-de-datos-en-rds)
4. [Subir la Base de Datos a Amazon RDS mediante pgAdmin](#subir-la-base-de-datos-a-amazon-rds-mediante-pgadmin)
5. [Modelo Relacional en PlantUML](#modelo-relacional-en-plantuml)
6. [Ejecución del Script para Crear Tablas y Funciones](#ejecución-del-script-para-crear-tablas-y-funciones)
7. [Script en Python para Cargar los CSV en las Tablas](#script-en-python-para-cargar-los-csv-en-las-tablas)
8. [Verificación de Datos Insertados](#verificación-de-datos-insertados)
9. [Configuración de la Función Lambda](#configuración-de-la-función-lambda)
10. [Creación de la API en API Gateway](#creación-de-la-api-en-api-gateway)

---

## Creación de la Base de Datos en Amazon RDS

1. Dirígete a la consola de Amazon RDS y haz clic en **"Base de datos"** en el panel izquierdo.  
    ![image.png](img_readme/image.png)

2. Haz clic en el botón **"Crear base de datos"**.  
    ![image.png](img_readme/image%201.png)

3. Selecciona la opción de **"Creación estándar"** y elige el motor **PostgreSQL**.  
    ![image.png](img_readme/image%202.png)

4. Asigna un nombre como identificador para la base de datos (RDS). Nota: Este no es el nombre de la base de datos que se configurará más adelante.  
    ![image.png](img_readme/image%203.png)

5. Selecciona la opción **Autoadministrado** y crea una contraseña para el acceso.  
    ![image.png](img_readme/image%204.png)

6. En la configuración de la instancia, selecciona el tipo **db.t4g.micro**.  
    ![image.png](img_readme/image%205.png)

7. Configura la opción para que **no se conecte a un recurso EC2**.  
    ![image.png](img_readme/image%206.png)

8. Configura la VPC y marca la opción para hacerla pública. Esto permitirá conectarte para crear tablas y cargar datos.  
    ![image.png](img_readme/image%207.png)  
    **Nota:** Dentro del grupo de seguridad de la VPC, crea una regla de entrada con el tipo **PostgreSQL**, protocolo **TCP**, puerto **5432**, y origen **cualquier dirección IP**. Esto será necesario para conectarte localmente y posteriormente con la función Lambda.  
    ![image.png](img_readme/image%208.png)

9. En el apartado de **"Configuración adicional"**, especifica el nombre de la base de datos.  
    ![image.png](img_readme/image%209.png)

10. Haz clic en **"Crear base de datos"** para finalizar.

---

## Conexión de pgAdmin a la Instancia de RDS

1. Abre **pgAdmin** en tu computadora.
2. En la barra lateral de **pgAdmin**, haz clic derecho sobre **Servers** y selecciona **Create** -> **Server...**.
3. Configura el servidor:
    - En la pestaña **General**, asigna un nombre para la conexión (por ejemplo, "RDS-PostgreSQL").
    - En la pestaña **Connection**:
      - En **Host name/address**, ingresa el **endpoint** de tu base de datos RDS (disponible en la consola de RDS de AWS).
      - En **Port**, usa el puerto `5432` (por defecto para PostgreSQL).
      - En **Maintenance database**, ingresa el nombre de tu base de datos.
      - En **Username**, ingresa el usuario que creaste para la base de datos de RDS.
      - En **Password**, ingresa la contraseña configurada.
4. Haz clic en **Save** para guardar la conexión y conectar **pgAdmin** a tu base de datos en Amazon RDS.  
    ![image.png](img_readme/image%2010.png)  
    ![image.png](img_readme/image%2011.png)

5. Una vez conectado, abre un script y ejecuta el código correspondiente.

---

## Subir Tablas y Funciones a la Base de Datos en RDS

1. **Esquema SQL (DDL) para crear las tablas en PostgreSQL**  
    Define las tablas utilizando tipos de datos adecuados como *VARCHAR*, *INTEGER*, *DOUBLE PRECISION* y *DATE/TIME*.

2. **Scripts en Python para cargar los CSV en las tablas**  
    Utiliza bibliotecas como *pandas* y *SQLAlchemy* para leer los archivos CSV y realizar inserciones masivas en la base de datos.

3. **Funciones SQL en PL/pgSQL**  
    Implementa funciones auxiliares como:
    - Cálculo de la distancia entre dos puntos (usando la fórmula de Haversine).
    - Contar accidentes cercanos a un punto de interés.
    - Obtener la velocidad de diseño más cercana dentro de un rango específico.  
    **Nota:** Asegúrate de que el grupo de seguridad permita conexiones desde la IP de tu máquina local.

---

## Subir la Base de Datos a Amazon RDS mediante pgAdmin

Utiliza la interfaz gráfica de **pgAdmin** para ejecutar los scripts SQL que crean las tablas y funciones en tu base de datos.

---

## Modelo Relacional en PlantUML

A continuación, se presenta el modelo relacional generado en PlantUML:  
![diagrama.png](img_readme/diagrama.png)

---

## Ejecución del Script para Crear Tablas y Funciones

1. Abre la conexión a tu base de datos en **pgAdmin**.
2. Ejecuta el script `ExtraerDatosServicios/script_crear_tablas_bd.sql` disponible en el presente repositorio
---

## Script en Python para Cargar los CSV en las Tablas

1. Instala las dependencias necesarias:

     ```bash
     pip install pandas==1.3.3 sqlalchemy==1.4.23 psycopg2==2.9.1 haversine==2.5.1
     ```

2. Navega al directorio correspondiente:

     ```bash
     cd Fork_PIS_20_02_AGENTES_IA/ExtraerDatosServicios
     ```

3. Ejecuta el script `poblarPostgreSQL.py` para cargar los datos en la base de datos.

---

## Verificación de Datos Insertados

Ejecuta la siguiente consulta para verificar los datos cargados:

```sql
SELECT * FROM accidents_quito LIMIT 10;
```

---

## Configuración de la Función Lambda

1. Navega al directorio `ExtraerDatosServicios/lambdaDats` dentro del repositorio.
2. Modifica las variables de entorno en el archivo `lambda_function.py` según tu configuración.  
    ![image.png](img_readme/image%2013.png)

3. Empaqueta el contenido del directorio en un archivo ZIP.  
    ![image.png](img_readme/image%2014.png)

4. Crea una nueva función Lambda en la consola de AWS:
    - Selecciona **"Crear desde cero"**.
    - Asigna un nombre a la función.
    - Selecciona **Python 3.13** como entorno de ejecución y arquitectura **x86_64**.  
      ![image.png](img_readme/image%2015.png)

5. Configura el rol correspondiente para la función Lambda.  
    ![image.png](img_readme/image%2017.png)

6. Habilita la VPC y selecciona la subred y el grupo de seguridad que permita tráfico en el puerto **5432**.  
    ![image.png](img_readme/image%2018.png)

7. Carga el archivo ZIP empaquetado previamente.  
    ![image.png](img_readme/image%2020.png)

8. Guarda los cambios y haz clic en **Deploy**.  
    ![image.png](img_readme/image%2022.png)

9. Crea un evento JSON para probar la función y haz clic en **Probar**.  
    ![image.png](img_readme/image%2024.png)  
    La salida esperada debe ser similar a la siguiente:  
    ![image.png](img_readme/image%2025.png)

---

## **Crear la API en API Gateway**

1. En el servicio API Gateway, selecciona **Crear API**.

    ![Servicio_Gateway.png](img_readme/Servicio_Gateway.png)

2. Configura los parámetros:
    - Tipo de API: **Nueva API**.
    - Nombre de la API.
    - Tipo de punto de conexión: **Regional**.

    ![Crear_API_Gateway.PNG](img_readme/Crear_API_Gateway.png)

3. En el menú **Recurso**, selecciona **Crear recurso**.

    ![image.png](img_readme/image%2026.png)

4. Crea un método POST y configura los detalles:
    - Tipo de integración: **Función Lambda**.
    - Región y ARN de la función Lambda.

    ![Crear_Metodo_POST.PNG](img_readme/Crear_Metodo_POST.png)

5. Implementa la API para obtener la URL de invocación.

    ![Deploy_API_Gateway (1).PNG](img_readme/Deploy_API_Gateway_(1).png)

    Ejemplo de URL:

    ```plaintext
    https://{nombre-api}.execute-api.{region}.amazonaws.com/{stage-name}/{resource-path}
    ```

---

## **4. Crear la API en API Gateway**

1. En el servicio API Gateway, selecciona **Crear API**.

    ![Servicio_Gateway.png](img_readme/Servicio_Gateway.png)

2. Configura los parámetros:
    - Tipo de API: **Nueva API**.
    - Nombre de la API.
    - Tipo de punto de conexión: **Regional**.

    ![Crear_API_Gateway.PNG](img_readme/Crear_API_Gateway.png)

3. En el menú **Recurso**, selecciona **Crear recurso**.

    ![image.png](img_readme/image%2026.png)

4. Crea un método POST y configura los detalles:
    - Tipo de integración: **Función Lambda**.
    - Región y ARN de la función Lambda.

    ![Crear_Metodo_POST.PNG](img_readme/Crear_Metodo_POST.png)

5. Implementa la API para obtener la URL de invocación.

    ![Deploy_API_Gateway (1).PNG](img_readme/Deploy_API_Gateway_(1).png)

    Ejemplo de URL:

    ```plaintext
    https://{nombre-api}.execute-api.{region}.amazonaws.com/{stage-name}/{resource-path}
    ```

# Publication

If you use POLIDriving in your research, please cite it as follows.

@article{marcillo2024polidriving,<br>
  title={POLIDriving: A Public-Access Driving Dataset for Road Traffic Safety Analysis},<br>
  author={Marcillo, Pablo and Arciniegas-Ayala, Cristian and Valdivieso Caraguay, {\'A}ngel Leonardo and Sanchez-Gordon, Sandra and Hern{\'a}ndez-{\'A}lvarez, Myriam},<br>
  journal={Applied Sciences},
  volume={14},<br>
  number={14},<br>
  pages={6300},<br>
  year={2024},<br>
  publisher={MDPI}<br>
}

# Downloads

The size of POLIDriving is about 150 MB.

# Contact

For questions or suggestions, please contact pablo.marcillo@epn.edu.ec
