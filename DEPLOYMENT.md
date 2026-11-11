# 🌐 Guía de Despliegue - FuelRecorder

Guía completa para desplegar la aplicación en producción.

---

## 📋 Tabla de Contenidos

- [Arquitectura de Producción](#arquitectura-de-producción)
- [Base de Datos (Supabase)](#base-de-datos-supabase)
- [Backend (Render/Railway)](#backend-render--railway)
- [Frontends (Vercel)](#frontends-vercel)
- [Configuración de Dominio Personalizado](#configuración-de-dominio-personalizado)
- [Monitoreo y Mantenimiento](#monitoreo-y-mantenimiento)
- [Checklist de Pre-Despliegue](#checklist-de-pre-despliegue)

---

## 🏗️ Arquitectura de Producción

```
┌──────────────────────────────────────────────────────┐
│                 Usuarios Finales                     │
└────────────┬──────────────────────┬──────────────────┘
             │                      │
             │                      │
    ┌────────▼────────┐    ┌────────▼────────┐
    │  Technician App │    │  Admin Dashboard│
    │   (Vercel)      │    │   (Vercel)      │
    │  technician.app │    │  admin.app      │
    └────────┬────────┘    └────────┬────────┘
             │                      │
             └──────────┬───────────┘
                        │
               ┌────────▼────────┐
               │   Backend API   │
               │  Render/Railway │
               │   api.app.com   │
               └────────┬────────┘
                        │
               ┌────────▼────────┐
               │   PostgreSQL    │
               │   Supabase      │
               │   db.supabase.co│
               └─────────────────┘
```

---

## 🗄️ Base de Datos (Supabase)

### Paso 1: Crear Proyecto

1. Ir a [supabase.com](https://supabase.com)
2. Sign up con GitHub
3. "New Project"
4. Configuración:
   - **Project Name:** fuel-recorder
   - **Database Password:** Generar y guardar en password manager
   - **Region:** US East (más cercano) o el más cercano a tu ubicación
   - **Pricing Plan:** Free (hasta 500MB, suficiente)

### Paso 2: Configurar Esquema

1. Ir a **SQL Editor** (icono de terminal)
2. Copiar contenido de `backend/schema.sql`
3. Pegar y ejecutar
4. Verificar que se crearon las tablas:
   - Ir a **Table Editor**
   - Deben aparecer: `tecnicos`, `suministros`

### Paso 3: Crear Admin Inicial

```sql
-- Generar hash en https://bcrypt-generator.com/
-- Password seguro que recuerdes

INSERT INTO tecnicos (nombre, ficha, password_hash, rol)
VALUES (
  'Administrador Principal',
  'ADMIN001',
  '$2b$10$TuHashGeneradoAqui',
  'admin'
);
```

### Paso 4: Obtener Credenciales

Ir a **Project Settings** > **Database** > **Connection string**

Copiar valores:
```
Host: db.<project-ref>.supabase.co
Port: 5432
Database: postgres
User: postgres
Password: <tu_password>
```

### Paso 5: Configurar Row Level Security (Opcional pero recomendado)

```sql
-- Deshabilitar RLS para empezar (más fácil)
ALTER TABLE tecnicos DISABLE ROW LEVEL SECURITY;
ALTER TABLE suministros DISABLE ROW LEVEL SECURITY;

-- O habilitar con políticas si quieres seguridad adicional
```

---

## 🚀 Backend (Render / Railway)

### Opción A: Desplegar en Render

#### 1. Preparar Repositorio

Subir código a GitHub:
```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/tu-usuario/fuel-recorder.git
git push -u origin main
```

#### 2. Crear Servicio en Render

1. Ir a [render.com](https://render.com)
2. Sign up con GitHub
3. **New** > **Web Service**
4. Conectar repositorio de GitHub
5. Configuración:
   - **Name:** fuel-recorder-api
   - **Region:** Us East (Oregon)
   - **Branch:** main
   - **Root Directory:** backend
   - **Runtime:** Node
   - **Build Command:** `npm install`
   - **Start Command:** `npm start`
   - **Instance Type:** Free

#### 3. Configurar Variables de Entorno

En Render > Environment:

```env
NODE_ENV=production
DB_HOST=db.<project-ref>.supabase.co
DB_PORT=5432
DB_NAME=postgres
DB_USER=postgres
DB_PASSWORD=<password_supabase>
JWT_SECRET=<generar_con_openssl_rand_hex_32>
JWT_EXPIRATION=7d
PORT=5000
```

**Generar JWT_SECRET seguro:**
```bash
openssl rand -hex 32
# Copiar el resultado
```

#### 4. Deploy

1. Click **Deploy**
2. Esperar 2-5 minutos
3. Ver logs para confirmar: `🚀 Server running on port 5000`
4. Copiar URL del servicio (ej: `https://fuel-recorder-api.onrender.com`)

### Opción B: Desplegar en Railway

1. Ir a [railway.app](https://railway.app)
2. Sign up con GitHub
3. **New Project** > **Deploy from GitHub repo**
4. Seleccionar repositorio
5. Configurar variables de entorno igual que Render
6. Deploy automático

---

## 💻 Frontends (Vercel)

### Frontend Admin

#### 1. Preparar

En `frontend-admin/package.json`, verificar:
```json
{
  "proxy": "http://localhost:5000"
}
```

Crear `frontend-admin/.env.production`:
```env
REACT_APP_API_URL=https://tu-backend-api.onrender.com/api
```

#### 2. Deploy en Vercel

1. Ir a [vercel.com](https://vercel.com)
2. Sign up con GitHub
3. **Add New...** > **Project**
4. Importar repositorio
5. Configuración:
   - **Project Name:** fuel-recorder-admin
   - **Framework Preset:** Create React App
   - **Root Directory:** frontend-admin
   - **Build Command:** `npm run build` (default)
   - **Output Directory:** `build` (default)
   - **Install Command:** `npm install` (default)

6. **Environment Variables:**
   ```
   REACT_APP_API_URL=https://tu-backend-api.onrender.com/api
   ```

7. **Deploy**
8. Obtener dominio: `https://fuel-recorder-admin.vercel.app`

### Frontend Técnico (Mismo proceso)

1. Crear `frontend/.env.production`:
```env
REACT_APP_API_URL=https://tu-backend-api.onrender.com/api
```

2. Deploy en Vercel como nuevo proyecto:
   - **Project Name:** fuel-recorder-tech
   - **Root Directory:** frontend
   - Mismas variables de entorno

3. Obtener dominio: `https://fuel-recorder-tech.vercel.app`

---

## 🔧 Configuración de Dominio Personalizado

### En Vercel

1. Ir a Project Settings > **Domains**
2. **Add** tu dominio (ej: `admin.fuelrecorder.com`)
3. Configurar DNS en tu proveedor de dominio:
   - **Type:** CNAME
   - **Name:** admin
   - **Value:** cname.vercel-dns.com
4. Esperar propagación (hasta 48h, usualmente minutos)

### En Render

1. Ir to Settings > **Custom Domain**
2. **Add Custom Domain**
3. Configurar DNS:
   - **Type:** CNAME
   - **Name:** api
   - **Value:** tu-app.onrender.com
4. Esperar propagación

---

## 📊 Monitoreo y Mantenimiento

### Backend (Render)

**Logs:**
- Dashboard > Logs
- Ver requests en tiempo real
- Filtrar por nivel (info, error, warn)

**Metrics:**
- Dashboard > Metrics
- CPU, Memory, Response Time
- Configurar alertas

**Health Check:**
```bash
curl https://tu-api.onrender.com/api/health
```

### Frontend (Vercel)

**Analytics:**
- Dashboard > Analytics
- Page views, load time
- Error tracking

**Logs:**
- Dashboard > Logs
- Build logs y runtime logs

### Base de Datos (Supabase)

**Dashboard:**
- Project > Dashboard
- Active connections, storage used
- API requests count

**Logs:**
- Project > Logs
- Database logs y query logs

### Backups Automáticos

**Supabase (Free tier):**
- Backups automáticos cada 24h
- Retención de 7 días
- Restaurar desde Dashboard

**Recomendación - Backup manual semanal:**
```bash
# Exportar base de datos
pg_dump -h db.<ref>.supabase.co -U postgres fuel_recorder > backup_$(date +%Y%m%d).sql

# Para restaurar:
psql -h db.<ref>.supabase.co -U postgres fuel_recorder < backup_20260411.sql
```

---

## ✅ Checklist de Pre-Despliegue

### Antes de Deploy

- [ ] Cambiar `JWT_SECRET` en backend `.env`
- [ ] Cambiar contraseñas por defecto
- [ ] Eliminar datos de prueba
- [ ] Probar localmente con variables de producción
- [ ] Verificar CORS configurado correctamente
- [ ] Crear usuario admin inicial
- [ ] Ejecutar todas las migraciones de BD
- [ ] Probar endpoints con Postman/Insomnia
- [ ] Build de frontends sin errores
- [ ] Probar flujos completos (login, crear, editar, eliminar)

### Seguridad

- [ ] HTTPS habilitado (automático en Vercel/Render)
- [ ] CORS restringido a dominios específicos
- [ ] Rate limiting implementado
- [ ] Helmet.js para security headers
- [ ] Input validation en todos los endpoints
- [ ] Passwords hasheados con bcrypt
- [ ] Tokens JWT con expiración
- [ ] No exponer variables sensibles en frontend

### Performance

- [ ] Frontends minificados y optimizados
- [ ] Imágenes comprimidas
- [ ] CDN configurado (automático en Vercel)
- [ ] Database indexes creados
- [ ] Connection pooling configurado

### Monitoreo

- [ ] Logs habilitados en todos los servicios
- [ ] Health check endpoint funcional
- [ ] Alertas de error configuradas
- [ ] Uptime monitoring (UptimeRobot, Pingdom)
- [ ] Backup schedule configurado

---

## 🔧 Post-Despliegue

### 1. Verificación Final

```bash
# Test backend
curl https://tu-api.onrender.com/api/health

# Test frontends
curl https://tu-admin.vercel.app
curl https://tu-tech.vercel.app
```

### 2. Crear Primer Usuario Real

1. Login en admin dashboard con ADMIN001
2. Crear técnicos reales
3. Probar login de técnico en app
4. Crear suministro de prueba
5. Verificar que aparece en admin

### 3. Configurar DNS (Opcional)

```
# En tu proveedor de dominio:

Type: A
Name: @
Value: 76.76.21.21 (Vercel IP)

Type: CNAME
Name: api
Value: tu-api.onrender.com
```

### 4. Configurar SSL

- Automático en Vercel y Render
- Verificar que HTTPS está activo
- Redirect HTTP > HTTPS automático

### 5. Documentar

- URLs de producción
- Credenciales de admin (en password manager)
- Variables de entorno (en lugar seguro)
- Contactos de soporte

---

## 🚨 Troubleshooting en Producción

### Backend no responde

```bash
# Ver logs en Render
Dashboard > Logs

# Verificar variables de entorno
Dashboard > Environment

# Restart manual
Dashboard > Manual Deploy > Clear build cache and deploy
```

### Frontend muestra errores

```bash
# Ver consola del navegador (F12)

# Ver build logs en Vercel
Dashboard > Functions > Build Logs

# Verificar API URL en .env.production
REACT_APP_API_URL=https://tu-api.onrender.com/api
```

### Error de Base de Datos

```bash
# Ver logs en Supabase
Project > Logs > Database

# Verificar conexión
Dashboard > Settings > Database > Connection info

# Test connection
psql -h db.<ref>.supabase.co -U postgres -d postgres
```

### CORS Error

En backend `server.js`:
```javascript
app.use(cors({
  origin: [
    'https://tu-admin.vercel.app',
    'https://tu-tech.vercel.app',
  ],
  credentials: true,
}));
```

---

## 📈 Escalamiento (Si es necesario)

### Cuando upgradear de plan

**Supabase:**
- > 500MB storage → Pro plan ($25/mes)
- > 2GB bandwidth → Pro plan
- Necesita backups más largos

**Render:**
- Respuesta lenta → Starter ($7/mes)
- Necesita más RAM → Standard ($12/mes)
- > 100 requests/segundo → Pro

**Vercel:**
- > 100GB bandwidth → Pro ($20/mes)
- Necesita SSR → Pro plan
- Team collaboration → Pro

---

## 💰 Costo Estimado Mensual

### Plan Gratuito (Recomendado para empezar)
- **Supabase:** $0 (500MB, 2GB bandwidth)
- **Render:** $0 (750 horas/mes, se duerme tras inactividad)
- **Vercel:** $0 (100GB bandwidth, ilimitados proyectos)
- **Total:** **$0/mes** ✨

### Plan Profesional (Producción seria)
- **Supabase Pro:** $25/mes
- **Render Starter:** $7/mes
- **Vercel Pro:** $20/mes (opcional)
- **Total:** **~$32-52/mes**

---

## 🎓 Mejores Prácticas

✅ **Siempre:**
- Usar variables de entorno, nunca hardcodear
- Mantener secretos seguros (1Password, LastPass)
- Backups regulares de BD
- Monitorear logs semanalmente
- Actualizar dependencias mensualmente

❌ **Nunca:**
- Subir `.env` a GitHub
- Usar credenciales de producción en desarrollo
- Ignorar warnings de seguridad
- Deploy en viernes por la tarde 😅
- Olvidar de monitorear errores

---

## 📞 Soporte Post-Despliegue

### Recursos

- **Render Docs:** [render.com/docs](https://render.com/docs)
- **Vercel Docs:** [vercel.com/docs](https://vercel.com/docs)
- **Supabase Docs:** [supabase.com/docs](https://supabase.com/docs)
- **PostgreSQL Docs:** [postgresql.org/docs](https://www.postgresql.org/docs/)

### Monitoreo Gratuito

- **UptimeRobot:** uptimerobot.com (cada 5 min)
- **Better Stack:** betterstack.com (logs y alerts)
- **Sentry:** sentry.io (error tracking, free tier)

---

**¡Tu aplicación está lista para producción! 🎉**
