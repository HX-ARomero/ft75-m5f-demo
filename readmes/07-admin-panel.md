# Panel de Administrador & AWS S3

[Volver](../README.md)

## Dependencias

```bash
npm install @aws-sdk/client-s3
npm install @aws-sdk/s3-request-presigner
npm install @vercel/node
```

## Variables de Entorno

```.env
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_access_key
AWS_REGION=aws_region
S3_BUCKET=your_s3_bucket
```

## Configuración de CORS en S3

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["PUT", "GET"],
    "AllowedOrigins": ["http://localhost:3000"],
    "ExposeHeaders": []
  }
]
```

## Configuración de "Políticas de bucket

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadImages",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::demo-ecommerce-admin-panel/*"
    }
  ]
}
```

## Reglas de Firestone (Firestone Rules)

- Podemos endurecerlas y utilizar las siguientes:

```ts
rules_version = '2';

service cloud.firestore {
	match /databases/{database}/documents {

	// PRODUCTS
	match /products/{productId} {

		// lectura pública
		allow read: if false;

		// solo admins pueden escribir
		allow write: if isAdmin();
	}

	// USERS
	match /users/{userId} {

		// cada usuario solo lee su perfil
		allow read:
			if request.auth != null
			&& request.auth.uid == userId;

		// nadie escribe directamente
		allow write: if false;
	}

	// ADMIN CHECK
	function isAdmin() {
		return request.auth != null
			&& get(
				/databases/$(database)/documents/users/$(request.auth.uid)
			).data.role == "admin";
		}
	}
}
```

---

# AWS S3 Presigned URL + Variables de Entorno + Firebase

## 1. ¿Qué es una Presigned URL en AWS S3?

Una **Presigned URL** es una URL temporal firmada digitalmente que permite realizar una operación específica sobre un archivo en S3 sin exponer credenciales de AWS al cliente.

Ejemplo:

```text
Frontend → solicita URL temporal al Backend
Backend → genera URL firmada usando credenciales AWS
Frontend → sube archivo directamente a S3 usando esa URL
```

### Flujo

```text
Usuario
  ↓
Selecciona imagen

Frontend (React)
  ↓
Solicita URL firmada al Backend (/api/presign)

Backend (Vercel Serverless Function)
  ↓
Usa credenciales AWS para generar URL temporal

AWS S3
  ↓
Devuelve Presigned URL

Frontend
  ↓
Hace PUT directamente a S3 con fetch()

AWS S3
  ↓
Guarda imagen

Frontend
  ↓
Obtiene publicUrl y la guarda en Firestore
```

### Ventaja

#### Sin Presigned URL

```text
Frontend → AWS usando AccessKey + SecretKey
```

Problema:

- Credenciales quedan expuestas
- Cualquier usuario puede acceder al bucket

#### Con Presigned URL

```text
Frontend → pide permiso temporal al Backend
Backend → firma operación específica
Frontend → usa URL temporal
```

Ventajas:

- Credenciales nunca salen del servidor
- Mayor seguridad
- Se controla tipo de archivo, tiempo y destino

---

## 2. Variables de Entorno en AWS

AWS requiere credenciales para acceder a S3.

Ejemplo:

```env
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION
S3_BUCKET
```

### ¿Por qué usar variables de entorno?

Porque **NO debemos escribir credenciales sensibles dentro del código fuente**.

#### Incorrecto

```ts
const secret = "AKIAXXXXXXXXXXX";
```

Problemas:

- Se sube a GitHub
- Cualquiera puede robar la clave
- Riesgo de facturación inesperada

#### Correcto

```ts
process.env.AWS_SECRET_ACCESS_KEY;
```

Las variables se guardan en:

- :contentReference[oaicite:0]{index=0} Dashboard
- Archivo `.env.local` (desarrollo local)

Ejemplo:

```env
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=xxxxxxxx
AWS_SECRET_ACCESS_KEY=xxxxxxxx
S3_BUCKET=demo-ecommerce-admin-panel
```

---

## 3. ¿Por qué Firebase es diferente?

Ejemplo:

```ts
const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
};
```

Puede parecer una credencial, pero **NO lo es**.

Importante:

```text
Firebase API Key NO es una clave privada
```

Se usa solamente para identificar qué proyecto usar.

Ejemplo:

Tu app React necesita saber:

```text
"Conectar al proyecto Firebase X"
```

La API Key simplemente identifica el proyecto.

---

## 4. Diferencia entre AWS Credentials y Firebase Config

### AWS Credentials

```env
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

Características:

- Son secretas
- Dan permisos reales sobre AWS
- Permiten crear, borrar o modificar recursos
- Nunca deben llegar al frontend

Ejemplo:

Si un atacante obtiene tu SecretKey:

Puede:

- Borrar bucket
- Subir archivos
- Generar costos

---

### Firebase Config

```ts
apiKey;
authDomain;
projectId;
```

Características:

- Son públicas
- Van en frontend
- Solo identifican proyecto Firebase
- No permiten control administrativo completo

Ejemplo:

Si alguien obtiene tu apiKey:

No puede:

- Borrar Firestore
- Administrar proyecto

Porque Firebase sigue validando:

- Authentication Rules
- Firestore Rules
- Security Rules

---

## 5. Arquitectura Comparada

### Firebase

```text
Frontend (React)
  ↓
Usa firebaseConfig pública
  ↓
Se conecta directamente a Firebase
```

Seguridad controlada por:

- Authentication
- Firestore Rules

---

### AWS S3 con Presigned URL

```text
Frontend (React)
  ↓
Pide URL temporal a Backend

Backend (Vercel Function)
  ↓
Usa SecretKey privada

AWS S3
  ↓
Genera URL temporal

Frontend
  ↓
Sube archivo usando URL firmada
```

Seguridad controlada por:

- SecretKey privada
- Expiración URL
- Restricciones Backend

---

## 6. ¿Por qué no subir directamente a S3 desde React?

Si hacemos esto:

```text
Frontend
  ↓
AWS SDK
  ↓
AccessKey + SecretKey en React
```

Problema:

Las credenciales quedan visibles en DevTools.

Cualquier usuario podría:

- Subir archivos
- Eliminar archivos
- Usar tu cuenta AWS

Por eso usamos:

```text
Presigned URL
```

Arquitectura correcta:

```text
Frontend → Backend → AWS → Frontend → Upload
```

---

## 7. Resumen rápido

### Firebase

- Configuración pública
- Puede ir en frontend
- Seguridad en Firestore/Auth Rules

### AWS S3

- Credenciales privadas
- Nunca van en frontend
- Se usan en backend/serverless functions

### Presigned URL

- Permite subir archivos temporalmente
- No expone credenciales AWS
- El frontend sube directo a S3

---

### Regla general

```text
Firebase Config → pública

AWS Secret Keys → privadas
```

---

## Arquitectura Final

```text
Usuario
  ↓
Selecciona imagen

React Frontend
  ↓
Solicita Presigned URL

Vercel API (/api/presign)
  ↓
Usa variables privadas AWS

AWS S3
  ↓
Genera URL temporal

React
  ↓
Hace PUT a S3

AWS S3
  ↓
Guarda archivo

React
  ↓
Guarda publicUrl en Firestore

Firebase
  ↓
Producto queda persistido con imagen
```

---

[Volver](../README.md)
