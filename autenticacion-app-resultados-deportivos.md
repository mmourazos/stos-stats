# Autenticación de usuarios: app de resultados deportivos

API en **Go**, frontend web, **sesiones con cookie HttpOnly** y **TLS** en todas las conexiones.

---

## 1. Requisitos y decisiones

### Roles

Solo se almacenan dos roles en la base de datos (`user` y `admin`). El **visitante** no es un rol: es una petición sin autenticar.

| Acción | Visitante | Usuario | Admin |
|---|---|---|---|
| `GET` resultados | ✅ | ✅ | ✅ |
| `GET` datos/perfiles públicos de usuarios | ✅ | ✅ | ✅ |
| `POST` resultado nuevo | ❌ | ✅ (queda como propietario) | ✅ |
| `PUT/DELETE` resultado | ❌ | Solo los suyos | Todos |
| Editar su propio perfil | ❌ | ✅ | ✅ |
| Gestionar usuarios y roles | ❌ | ❌ | ✅ |

La **lectura es completamente pública**; la autenticación solo se necesita para **escribir**.

### Decisiones tomadas

- Sesiones opacas en servidor con **cookie HttpOnly** (no JWT).
- **TLS** obligatorio en todas las conexiones.
- Lectura pública de los datos de todos los usuarios (perfil público, sin datos sensibles).

---

## 2. Stack y estructura

- **Router**: `net/http` con `ServeMux` (Go 1.22+ admite `"GET /results/{id}"`) o `chi`.
- **Base de datos**: PostgreSQL con `pgx` y, opcionalmente, `sqlc`.
- **Migraciones**: `golang-migrate`.
- **Hash de contraseñas**: `golang.org/x/crypto/argon2` o `github.com/alexedwards/argon2id`.

```
/cmd/api/main.go
/internal/auth/      (hash, sesiones, middleware)
/internal/users/     (handlers y repositorio)
/internal/results/   (handlers y repositorio)
/migrations/
```

---

## 3. Modelo de datos

```sql
CREATE TABLE users (
  id            BIGSERIAL PRIMARY KEY,
  email         TEXT UNIQUE NOT NULL,
  display_name  TEXT UNIQUE NOT NULL,      -- alias público
  password_hash TEXT NOT NULL,
  role          TEXT NOT NULL DEFAULT 'user' CHECK (role IN ('user','admin')),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sessions (
  token_hash BYTEA PRIMARY KEY,            -- SHA-256 del token
  user_id    BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  expires_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE results (
  id       BIGSERIAL PRIMARY KEY,
  owner_id BIGINT NOT NULL REFERENCES users(id),
  -- ...campos del deporte (equipos, marcador, fecha, etc.)
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- `owner_id` es la clave para "actualizar solo sus propios datos".
- Nunca se guardan contraseñas ni tokens en claro: solo sus hashes.

---

## 4. Despliegue: mismo *site*

El diseño más sencillo y seguro es que frontend y API estén en el mismo site:

- **Opción A**: un dominio, con proxy inverso que sirve el frontend en `/` y la API en `/api`. Sin CORS.
- **Opción B**: `app.midominio.com` y `api.midominio.com`. Same-site, pero requiere CORS con credenciales.

Evita dominios totalmente distintos: obligan a `SameSite=None`, hacen imprescindible la protección CSRF y dan problemas con cookies de terceros.

---

## 5. Sesiones con cookie HttpOnly

### 5.1 Generar y guardar el token

El token es opaco y aleatorio. En la base de datos solo se guarda su hash.

```go
func newToken() (token string, hash []byte, err error) {
    b := make([]byte, 32)
    if _, err = rand.Read(b); err != nil {
        return "", nil, err
    }
    token = base64.RawURLEncoding.EncodeToString(b)
    return token, hashToken(token), nil
}

func hashToken(t string) []byte {
    s := sha256.Sum256([]byte(t))
    return s[:]
}
```

SHA-256 basta aquí (no Argon2) porque el token ya tiene 256 bits de entropía.

### 5.2 Store de sesiones

```go
const sessionTTL = 7 * 24 * time.Hour

func (s *Store) Create(ctx context.Context, userID int64) (string, time.Time, error) {
    token, h, err := newToken()
    if err != nil {
        return "", time.Time{}, err
    }
    exp := time.Now().Add(sessionTTL)
    _, err = s.db.Exec(ctx,
        `INSERT INTO sessions (token_hash, user_id, expires_at) VALUES ($1,$2,$3)`,
        h, userID, exp)
    return token, exp, err
}

func (s *Store) UserByToken(ctx context.Context, token string) (*User, error) {
    var u User
    err := s.db.QueryRow(ctx, `
        SELECT u.id, u.role, u.display_name
        FROM sessions s JOIN users u ON u.id = s.user_id
        WHERE s.token_hash = $1 AND s.expires_at > now()`,
        hashToken(token)).Scan(&u.ID, &u.Role, &u.DisplayName)
    return &u, err
}

func (s *Store) Delete(ctx context.Context, token string) error {
    _, err := s.db.Exec(ctx, `DELETE FROM sessions WHERE token_hash = $1`, hashToken(token))
    return err
}

func (s *Store) DeleteAllForUser(ctx context.Context, userID int64) error {
    _, err := s.db.Exec(ctx, `DELETE FROM sessions WHERE user_id = $1`, userID)
    return err
}
```

### 5.3 La cookie

```go
func setSessionCookie(w http.ResponseWriter, token string, exp time.Time, secure bool) {
    http.SetCookie(w, &http.Cookie{
        Name:     "__Host-session",
        Value:    token,
        Path:     "/",
        Expires:  exp,
        HttpOnly: true,
        Secure:   secure,
        SameSite: http.SameSiteLaxMode,
    })
}

func clearSessionCookie(w http.ResponseWriter, secure bool) {
    http.SetCookie(w, &http.Cookie{
        Name: "__Host-session", Value: "", Path: "/",
        MaxAge: -1, HttpOnly: true, Secure: secure, SameSite: http.SameSiteLaxMode,
    })
}
```

- El prefijo `__Host-` obliga al navegador a exigir `Secure`, `Path=/` y ausencia de `Domain`, de modo que un subdominio comprometido no pueda pisar la cookie.
- `secure` debe ser `true` en producción. Con `mkcert` en desarrollo puedes mantenerlo también en local.
- `Strict` es más restrictivo, pero con `Lax` los enlaces externos a tu web mantienen la sesión.

### 5.4 Reglas del ciclo de vida

- **Caducidad absoluta** de 7 a 30 días según la sensibilidad. Para caducidad por inactividad, añade `last_seen_at` y renueva `expires_at` como mucho una vez cada cierto tiempo (p. ej., cada hora).
- **Cambio de contraseña o de rol**: llama a `DeleteAllForUser` (o a todas menos la actual).
- **Borrado de cuenta**: `ON DELETE CASCADE` limpia las sesiones.
- **Limpieza**: una goroutine con `time.Ticker` que ejecute `DELETE FROM sessions WHERE expires_at < now()` cada hora.

---

## 6. Endpoints de autenticación

- `POST /auth/register`: valida email y contraseña (mínimo 10 caracteres, mejor comprobar contra listas de contraseñas filtradas que imponer reglas de símbolos), hashea con Argon2id y crea el usuario con `role='user'`. **El rol nunca se lee del body.**
- `POST /auth/login`: busca el usuario, compara el hash (si no existe, compara contra un hash falso para igualar tiempos), responde con error genérico (`401 "credenciales inválidas"`), crea una sesión **nueva** y pone la cookie. No devuelve el token en el body.
- `POST /auth/logout`: borra la sesión y la cookie.
- `GET /auth/me`: devuelve el usuario actual (el frontend lo usa al cargar).
- Recuperación de contraseña (opcional): `POST /auth/forgot-password` y `POST /auth/reset-password` con token aleatorio de un solo uso, hash en BD, caducidad de 15 a 60 minutos y revocación de sesiones al cambiar la contraseña. Responde siempre igual exista o no el email.

El primer administrador se crea con un **comando de seed o una variable de entorno** al arrancar, nunca por la API pública.

```go
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    // 1. Limitar body y decodificar {email, password}
    // 2. Buscar usuario por email
    // 3. Comparar hash (hash falso si el usuario no existe)
    // 4. Error genérico: 401 "credenciales inválidas"
    // 5. Crear sesión NUEVA y poner la cookie
    token, exp, err := h.sessions.Create(r.Context(), user.ID)
    if err != nil { /* 500 */ }
    setSessionCookie(w, token, exp, h.cfg.SecureCookies)
    // 6. Devolver el usuario en JSON (sin el token)
}

func (h *AuthHandler) Logout(w http.ResponseWriter, r *http.Request) {
    if c, err := r.Cookie("__Host-session"); err == nil {
        _ = h.sessions.Delete(r.Context(), c.Value)
    }
    clearSessionCookie(w, h.cfg.SecureCookies)
    w.WriteHeader(http.StatusNoContent)
}
```

---

## 7. Middlewares

```go
type ctxKey struct{}

// Authenticate: si hay cookie válida, mete el usuario en el contexto.
// Si no la hay, deja pasar como visitante.
func (a *Auth) Authenticate(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if c, err := r.Cookie("__Host-session"); err == nil {
            if u, err := a.store.UserByToken(r.Context(), c.Value); err == nil {
                r = r.WithContext(context.WithValue(r.Context(), ctxKey{}, u))
            }
        }
        next.ServeHTTP(w, r)
    })
}

// RequireRole: 401 si no hay usuario, 403 si el rol no está permitido.
func RequireRole(roles ...string) func(http.Handler) http.Handler { /* ... */ }
```

Si la cookie existe pero es inválida o está caducada, trátala como visitante y bórrala en la respuesta.

### Orden recomendado

```
Recover → Logging → Cabeceras de seguridad (HSTS, CSP...) → RateLimit → CSRF/CrossOrigin → Authenticate → router (+ RequireRole por ruta)
```

---

## 8. Rutas según los permisos

```go
// Públicas (sin middleware de rol)
mux.HandleFunc("GET /results", h.ListResults)
mux.HandleFunc("GET /results/{id}", h.GetResult)
mux.HandleFunc("GET /users", h.ListUsers)
mux.HandleFunc("GET /users/{id}", h.GetUser)
mux.HandleFunc("GET /users/{id}/results", h.ListUserResults)

// Solo usuarios autenticados
user := RequireRole("user", "admin")
mux.Handle("POST /results", user(http.HandlerFunc(h.CreateResult)))
mux.Handle("PUT /results/{id}", user(http.HandlerFunc(h.UpdateResult)))
mux.Handle("DELETE /results/{id}", user(http.HandlerFunc(h.DeleteResult)))
mux.Handle("PUT /users/me", user(http.HandlerFunc(h.UpdateMe)))

// Solo admin
admin := RequireRole("admin")
mux.Handle("PATCH /admin/users/{id}/role", admin(http.HandlerFunc(h.SetRole)))
mux.Handle("DELETE /admin/users/{id}", admin(http.HandlerFunc(h.DeleteUser)))
```

---

## 9. Comprobación de propiedad

El rol `user` no basta: hay que asegurar que edita solo lo suyo. Lo más robusto es hacerlo en la propia consulta:

```go
if u.Role == "admin" {
    err = repo.Update(ctx, id, data)
} else {
    err = repo.UpdateIfOwner(ctx, id, u.ID, data)  // WHERE id=$1 AND owner_id=$2
}
// Si no afecta a ninguna fila, responde 404 (o 403).
```

---

## 10. Datos públicos de usuarios

Que cualquiera pueda consultar a los usuarios no significa exponer todo.

**DTO público separado del modelo de BD.** Nunca devuelvas `email`, `password_hash`, `role` ni nada interno:

```go
type PublicUser struct {
    ID          int64     `json:"id"`
    DisplayName string    `json:"display_name"`
    CreatedAt   time.Time `json:"created_at"`
    // estadísticas deportivas, etc.
}
```

**Medidas asociadas:**

- **RGPD**: mostrar solo lo necesario (alias, no nombre real), informar en el registro de qué datos son públicos y permitir borrar la cuenta y los datos.
- **Paginación obligatoria** en `GET /users` y `GET /results` (`?limit=50&cursor=...`, con máximo de `limit` fijado en servidor).
- **Rate limiting también en lectura** (más generoso que en `/auth/*`) y valorar caché (`Cache-Control`, ETag o caché en memoria).
- **Evitar la enumeración de cuentas**: login y registro no deben revelar si un email existe.

---

## 11. Protección CSRF (obligatoria con cookies)

`SameSite=Lax` cubre mucho, pero no debe ser la única defensa.

- **Go 1.25 o superior**: `http.NewCrossOriginProtection()` en la librería estándar. Rechaza peticiones no seguras (`POST/PUT/PATCH/DELETE`) cuyo origen no sea el tuyo, usando `Sec-Fetch-Site` y `Origin`.
- **Alternativa**: exigir una cabecera personalizada (`X-Requested-With`) en las peticiones que modifican datos, o un token CSRF con doble envío de cookie.
- Nunca cambies datos con `GET`.

## 12. CORS (solo con la opción B)

- `Access-Control-Allow-Origin` con el **origen exacto** del frontend (lista blanca; no reflejes el `Origin` sin comprobar).
- `Access-Control-Allow-Credentials: true`.
- `Vary: Origin`.
- Con la opción A (proxy inverso) no necesitas CORS.

---

## 13. TLS

Con cookies de sesión y datos personales, TLS es un requisito. Las cookies `Secure` y el prefijo `__Host-` solo funcionan sobre HTTPS.

### 13.1 Dónde terminar TLS

| Opción | Cuándo |
|---|---|
| **TLS directo en Go** (`ListenAndServeTLS`) | Despliegue sencillo, un solo servidor, sin proxy. |
| **Proxy inverso (Caddy, nginx, Traefik)** | Si sirves frontend en `/` y API en `/api`. Caddy obtiene y renueva certificados solo. |

Si hay proxy, el tramo proxy → Go debe ir por una red privada o también cifrado.

### 13.2 Certificados

- **Producción**: Let's Encrypt con renovación automática (`github.com/caddyserver/certmagic` o `golang.org/x/crypto/acme/autocert`).
- **Desarrollo local**: `mkcert` para probar con `Secure` y `__Host-` sin quitarlos.
- **Nunca** versiones las claves privadas. Cárgalas desde ficheros con permisos `0600` o desde un gestor de secretos.

### 13.3 Servidor con TLS y timeouts

```go
func main() {
    mux := buildRouter() // tu cadena de middlewares

    tlsCfg := &tls.Config{
        MinVersion: tls.VersionTLS12, // TLS 1.3 si tus clientes lo permiten
    }

    srv := &http.Server{
        Addr:              ":443",
        Handler:           mux,
        TLSConfig:         tlsCfg,
        ReadHeaderTimeout: 5 * time.Second,
        ReadTimeout:       15 * time.Second,
        WriteTimeout:      30 * time.Second,
        IdleTimeout:       120 * time.Second,
    }

    log.Fatal(srv.ListenAndServeTLS("fullchain.pem", "privkey.pem"))
}
```

- HTTP/2 se activa solo con `ListenAndServeTLS`.
- Los timeouts protegen contra conexiones lentas (Slowloris); el servidor de Go no tiene ninguno por defecto.

### 13.4 Redirección HTTP → HTTPS

```go
go http.ListenAndServe(":80", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    http.Redirect(w, r, "https://"+r.Host+r.URL.RequestURI(), http.StatusMovedPermanently)
}))
```

Valida `r.Host` contra tu dominio para no redirigir a hosts arbitrarios.

### 13.5 HSTS

```go
w.Header().Set("Strict-Transport-Security", "max-age=63072000; includeSubDomains")
```

Empieza con un `max-age` bajo y añade `preload` solo cuando estés seguro, porque es difícil de revertir.

### 13.6 Si hay proxy delante

Acepta `X-Forwarded-Proto` y `X-Forwarded-For` **solo de las IP de tu proxy**; de lo contrario, un cliente puede falsificarlas y saltarse el rate limiting.

### 13.7 El resto de conexiones también

- **PostgreSQL**: `sslmode=verify-full` en la cadena de conexión de `pgx`. `require` cifra pero no verifica la identidad.
- **SMTP**: STARTTLS o TLS implícito.
- **Servicios externos**: nunca uses `InsecureSkipVerify: true`.

### 13.8 Operación

- Si usas ficheros de certificado, necesitas recarga sin reiniciar (`GetCertificate` con relectura periódica) y alerta de caducidad.
- Firewall: abre 443 (y 80 para redirección/ACME).
- Logs: no registres cabeceras `Cookie` ni `Set-Cookie`.

---

## 14. Otras medidas de seguridad

- **Rate limiting** en `/auth/*` (`golang.org/x/time/rate` o `httprate`), idealmente por IP y por email.
- **Límite del body** con `http.MaxBytesReader` y `json.Decoder.DisallowUnknownFields()`.
- **Consultas parametrizadas** siempre.
- **Secretos** en variables de entorno, nunca en el repositorio.
- **Cabeceras de seguridad**: HSTS, CSP, `X-Content-Type-Options`.
- Opcional: 2FA (TOTP) para administradores.
- Revisión periódica de dependencias y cumplimiento del RGPD.

---

## 15. Frontend

- Llamadas con `credentials: 'include'` (innecesario con la opción A, al ser mismo origen).
- Al arrancar, `GET /auth/me`: `200` es sesión activa con su rol; `401` es visitante.
- Las páginas de consulta (resultados, usuarios, perfiles) no necesitan ruta protegida.
- Se protegen solo las pantallas de crear/editar y el panel de administración.
- Los botones de editar/borrar solo se muestran si el usuario es propietario o admin (la seguridad real la impone la API).
- Ante un `401` en una acción protegida, redirige al login.
- No guardes nada de sesión en `localStorage`.

---

## 16. Tests

Con `httptest`, al menos:

**Permisos**
- Visitante: `GET /results`, `GET /users` y `GET /users/{id}` → 200, y la respuesta **no contiene** `email` ni `password_hash`.
- Visitante: `POST /results` y `PUT /users/me` → 401.
- Usuario: edita lo suyo OK; lo ajeno → 403/404; `/admin/*` → 403.
- Admin: acceso total.
- Registro con `"role":"admin"` en el body → se ignora.
- `GET /users?limit=100000` → se recorta al máximo permitido.

**Sesión y cookie**
- La respuesta de login lleva `Set-Cookie` con `HttpOnly`, `Secure` y `SameSite`, y el body no contiene el token.
- Tras logout, la misma cookie devuelve `401`.
- Cookie caducada o manipulada → se trata como visitante.
- Dos logins generan tokens distintos.
- El cambio de contraseña invalida las demás sesiones.
- `POST` con `Origin` ajeno → rechazado.

**TLS**
- Con `httptest.NewTLSServer`, las cookies llevan `Secure`.
- HTTP → HTTPS responde `301`.
- Tras el despliegue: pasar SSL Labs o `testssl.sh` (solo TLS 1.2/1.3, cadena completa).

---

## 17. Checklist de implementación

1. Definir roles y permisos (`user`, `admin`; visitante = sin sesión).
2. Crear migraciones: `users`, `sessions`, `results`.
3. Implementar hash Argon2id y generación de tokens.
4. Implementar store de sesiones y limpieza periódica.
5. Implementar handlers `register`, `login`, `logout`, `me`.
6. Añadir middlewares: `Authenticate`, `RequireRole`, CSRF, rate limiting, cabeceras de seguridad.
7. Proteger rutas y comprobar propiedad en las consultas.
8. Exponer DTOs públicos con paginación.
9. Configurar TLS, redirección y HSTS.
10. Cifrar conexiones a la BD y otros servicios.
11. Integrar el frontend (`/auth/me`, rutas protegidas).
12. Escribir los tests y verificar con SSL Labs.
