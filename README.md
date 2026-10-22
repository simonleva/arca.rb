# arca.rb

🧾 Webservices de ARCA (ex AFIP) para Ruby.

Facturá, consultá el padrón, constatá comprobantes y leé la Ventanilla Electrónica desde Ruby, sin armar XML a mano.

**arca.rb** se autentica contra WSAA con tu certificado, guarda el ticket de acceso mientras sirve y te devuelve las respuestas SOAP como hashes de Ruby.

Para aplicaciones que hablan con ARCA: sistemas de facturación que piden CAE, apps que completan datos de un cliente a partir de su CUIT, procesos que verifican comprobantes recibidos o bajan las comunicaciones de la Ventanilla Electrónica.

**arca.rb** maneja:

- autenticación WSAA, con el ticket de acceso guardado en un archivo o en el backend que elijas
- facturación electrónica (WSFE), con CAE y CAEA
- factura electrónica de crédito MiPyMEs (WSFeCred)
- constatación de comprobantes (WSCDC)
- régimen de percepción de IVA (WSRGIVA)
- padrón de contribuyentes A4, A5 y A100, y constancia de inscripción
- Ventanilla Electrónica (VEConsumer), con adjuntos
- declaraciones aduaneras (WConsDeclaracion)
- un cliente SOAP de bajo nivel para cualquier otro WSDL
- homologación y producción, y WSDL locales para tus tests

> No arma el PDF de la factura ni calcula importes ni alícuotas. Lo que le pasás es lo que llega a ARCA.

## Contenido

- [Instalación](#instalación)
- [Certificado](#certificado)
- [Ambientes](#ambientes)
- [Autenticación](#autenticación)
- [Facturación electrónica](#facturación-electrónica)
- [Factura de crédito MiPyMEs](#factura-de-crédito-mipymes)
- [Constatación de comprobantes](#constatación-de-comprobantes)
- [Percepción de IVA](#percepción-de-iva)
- [Padrón](#padrón)
- [Constancia de inscripción](#constancia-de-inscripción)
- [Ventanilla Electrónica](#ventanilla-electrónica)
- [Declaraciones aduaneras](#declaraciones-aduaneras)
- [Errores](#errores)
- [Cliente de bajo nivel](#cliente-de-bajo-nivel)
- [Tests](#tests)
- [Historia](#historia)
- [Contribuir](#contribuir)
- [Agradecimientos](#agradecimientos)
- [Licencia](#licencia)

## Instalación

Agregá esta línea al Gemfile de tu aplicación:

```ruby
gem "arca.rb"
```

Requiere Ruby >= 3.3.4

## Certificado

```ruby
require "arca"

ws = Arca::WSFE.new(
  env: :development,
  cuit: "20123456789",
  key: File.read("ruta/a/clave-privada.key"),
  cert: File.read("ruta/a/certificado.crt")
)

ws.cotizacion("DOL") # => la cotización del dólar, como Float
```

Cada servicio necesita la clave privada y el certificado que ARCA emite para tu CUIT. Seguí los [pasos para solicitar el certificado](http://www.afip.gov.ar/ws/WSAA/cert-req-howto.txt): te queda un `.crt` y una clave privada (`.key` o `.pem`). Pasale el contenido de los archivos, no la ruta.

Todas las clases de servicio reciben las mismas opciones:

- **env** - `:development`, `:production` o `:test`, símbolo o string; por defecto `:test` (ver [Ambientes](#ambientes))
- **cuit** - la CUIT que representás
- **key** - la clave privada, en PEM
- **cert** - el certificado
- **savon** - opcional, un hash que se suma a las opciones del cliente [Savon](https://github.com/savonrb/savon), por ejemplo `savon: { open_timeout: 10, read_timeout: 30 }`
- **ta_path**, **store**, **ttl** - opcionales, para el ticket de acceso (ver [Autenticación](#autenticación))

Cada clase pide el ticket de WSAA para su propio servicio, así que el certificado tiene que estar autorizado en ARCA para cada uno que uses:

| Clase | Servicio en WSAA |
|-------|------------------|
| `Arca::WSFE` | `wsfe` |
| `Arca::WSFeCred` | `wsfecred` |
| `Arca::WSCDC` | `wscdc` |
| `Arca::WSRGIVA` | `wsrgiva` |
| `Arca::PersonaServiceA4` | `ws_sr_padron_a4` |
| `Arca::PersonaServiceA5` | `ws_sr_padron_a5` |
| `Arca::PersonaServiceA100` | `ws_sr_padron_a100` |
| `Arca::WSConstanciaInscripcion` | `ws_sr_constancia_inscripcion` |
| `Arca::VEConsumer` | `veconsumerws` |
| `Arca::WConsDeclaracion` | `wconsdeclaracion` |

## Ambientes

```ruby
Arca::WSFE.new(env: :production, cuit:, key:, cert:)
```

| `env` | Uso |
|-------|-----|
| `:development` | Homologación: los servidores de prueba de ARCA. |
| `:production` | Producción: los servidores reales de ARCA. |
| `:test` | Para tus tests: usa los WSDL de `test/fixtures` que trae la gema. |

Sin `env`, la clase usa `:test`. Para hablar con ARCA tenés que pasar `:development` o `:production`.

## Autenticación

```ruby
wsaa = Arca::WSAA.new(env: :development, cuit: "20123456789", key: key, cert: cert, service: "wsfe")
wsaa.auth # => { token: "...", sign: "..." }
```

Los servicios llaman a WSAA solos; lo usás directo solo si querés el token y el sign. `service` es por defecto `"wsfe"`.

El ticket de acceso (TA) se guarda hasta que vence y se reutiliza, porque ARCA rechaza pedir uno nuevo mientras el anterior sigue vigente. Por defecto va a `tmp/<cuit>-<env>-<servicio>-ta.json` dentro del directorio actual; `ta_path` cambia esa ruta y `ttl` los segundos de validez que se piden (2400 por defecto).

Para guardarlo en otro lado, por ejemplo cifrado en tu base y uno por cliente, pasá un `store`:

```ruby
class TicketStore
  def read(cuit:, env:, service:)
    # nil, o un Hash con :token, :sign, :generation_time y :expiration_time
  end

  def write(cuit:, env:, service:, ta:, expires_at:)
    # ta es el Hash del ticket; expires_at, un Time
  end
end

Arca::WSFE.new(env: :production, cuit:, key:, cert:, store: TicketStore.new)
```

`read` puede devolver las fechas como `Time` o como string ISO 8601. `write` se llama después de cada login exitoso. Si `write` levanta una excepción, llega como `Arca::ServerError`.

## Facturación electrónica

```ruby
ws = Arca::WSFE.new(env: :development, cuit: "20123456789", key: key, cert: cert)

ultimo = ws.ultimo_comprobante_autorizado(pto_vta: 2, cbte_tipo: 1)

ws.autorizar_comprobantes(cbte_tipo: 1, pto_vta: 2, comprobantes: [ {
  cbte_nro: ultimo + 1, concepto: 1, doc_tipo: 80, doc_nro: 30521189203,
  cbte_fch: Date.today, imp_total: 1270.48, imp_neto: 1049.98, imp_iva: 220.50,
  mon_id: "PES", mon_cotiz: 1,
  iva: { alic_iva: [ { id: 5, base_imp: 1049.98, importe: 220.50 } ] }
} ])
# => [{ cae: "...", cae_fch_vto: #<Date ...>, cbte_nro: ..., resultado: "A", observaciones: [] }]
```

Un comprobante rechazado no levanta una excepción: vuelve con `resultado: "R"` y el motivo en `observaciones`, como `[{ code: 10048, msg: "..." }]`.

Las fechas van y vuelven como `Date`. Las claves en snake_case se convierten a las de ARCA (`cbte_fch` pasa a `CbteFch`).

```ruby
ws.consultar_comprobante(pto_vta: 2, cbte_tipo: 1, cbte_nro: 1)
ws.cant_max_registros_x_lote

ws.solicitar_caea                       # el CAEA de la próxima quincena
ws.consultar_caea(Date.today)
ws.informar_comprobantes_caea(cbte_tipo: 1, pto_vta: 2, comprobantes: [ { caea: "21043476341977", cbte_nro: 1 } ])
ws.informar_caea_sin_movimientos("21043476341977", 2)
ws.consultar_caea_sin_movimientos("21043476341977", 2)
```

Si el CAEA de la quincena ya estaba otorgado, `solicitar_caea` lo consulta y te lo devuelve.

Parámetros y conectividad:

```ruby
ws.dummy
ws.cotizacion("DOL")
ws.tipos_comprobantes
ws.tipos_documentos
ws.tipos_concepto
ws.tipos_monedas
ws.tipos_iva
ws.tipos_tributos
ws.tipos_opcional
ws.tipos_condicion_iva_receptor(clase_cmp: "A")
ws.actividades
ws.puntos_venta
```

## Factura de crédito MiPyMEs

```ruby
ws = Arca::WSFeCred.new(env: :development, cuit: "20123456789", key: key, cert: cert)

ws.consultar_ctas_ctes(rol_cuit_representada: "Receptor")
ws.consultar_cta_cte(cod_cta_cte: 12345)

ws.rechazar_f_e_cred({ cod_cta_cte: 12345 },
  array_motivos_rechazo: [ { cod_motivo: 1, desc_motivo: "...", justificacion: "..." } ])
```

La cuenta corriente se identifica con `{ cod_cta_cte: }` o con `{ id_factura: { cuit_emisor:, cod_tipo_cmp:, pto_vta:, nro_cmp: } }`. Un comprobante, con `{ cuit_emisor:, cod_tipo_cmp:, pto_vta:, nro_cmp: }`.

Operaciones:

- `aceptar_f_e_cred(id_cta_cte, **opciones)`
- `rechazar_f_e_cred(id_cta_cte, array_motivos_rechazo:)`
- `rechazar_nota_dc(id_comprobante, array_motivos_rechazo:)`
- `informar_factura_agt_dpto_cltv(id_cta_cte, cta_agente:)`
- `informar_cancelacion_total_f_e_cred(id_cta_cte, array_formas_cancelacion:, importe_cancelacion:)`
- `modificar_opcion_transferencia(id_cta_cte, opcion_transferencia:)`, con `"ADC"` o `"SCA"`

Consultas:

- `consultar_comprobantes(rol_cuit_representada:, ...)`, con `"Emisor"` o `"Receptor"` y filtros opcionales
- `consultar_ctas_ctes(rol_cuit_representada:, ...)`
- `consultar_cta_cte(id_cta_cte)`
- `consultar_cuentas_en_agt_dpto_cltv`
- `consultar_facturas_agt_dpto_cltv(id_cta_cte: nil, filtro_fecha: nil)`
- `consultar_monto_obligado_recepcion(cuit_consultada:, fecha_emision:)`
- `consultar_historial_estados_comprobante(id_comprobante)`
- `consultar_historial_estados_cta_cte(id_cta_cte)`
- `obtener_remitos(id_comprobante)`
- `consultar_tipos_retenciones`, `consultar_tipos_motivos_rechazo`, `consultar_tipos_formas_cancelacion`, `consultar_tipos_ajustes_operacion`
- `dummy`

## Constatación de comprobantes

```ruby
ws = Arca::WSCDC.new(env: :development, cuit: "20123456789", key: key, cert: cert)

ws.comprobante_constatar(
  cbte_modo: "CAE",
  cuit_emisor: 20000000001,
  pto_vta: 1,
  cbte_tipo: 1,
  cbte_nro: 2,
  cbte_fch: "20250101",
  imp_total: 100.0,
  cod_autorizacion: "61023008595705"
)
# => { resultado: "A", cmp_resp: {...}, observaciones: [], fch_proceso: "...", errors: [], events: [] }
```

`cbte_modo` es `"CAE"`, `"CAEA"` o `"CAI"`. También acepta `doc_tipo_receptor` y `doc_nro_receptor`.

Parámetros: `comprobantes_modalidad_consultar`, `comprobantes_tipo_consultar`, `documentos_tipo_consultar`, `opcionales_tipo_consultar` y `dummy`.

## Percepción de IVA

```ruby
ws = Arca::WSRGIVA.new(env: :development, cuit: "20123456789", key: key, cert: cert)

ws.consultar_constancia_por_lote([ "20123456789", "20999888777" ])
# => una constancia por CUIT, con vigencia y leyenda, o con codigo_error y descripcion_error
```

Hasta 100 CUIT por llamada. También tiene `dummy`.

## Padrón

```ruby
padron = Arca::PersonaServiceA5.new(env: :development, cuit: "20123456789", key: key, cert: cert)

padron.get_persona("20123456789")
padron.get_persona_list([ "20123456789", "20999888777" ])
```

`Arca::PersonaServiceA4` tiene el mismo `get_persona`, sin `get_persona_list`.

`Arca::PersonaServiceA100` devuelve las tablas de parámetros del padrón:

```ruby
a100 = Arca::PersonaServiceA100.new(env: :development, cuit: "20123456789", key: key, cert: cert)

a100.jurisdictions    # provincias
a100.company_types    # tipos de empresa jurídica
a100.public_organisms # organismos informantes
```

Las tres tienen `dummy`.

## Constancia de inscripción

```ruby
ws = Arca::WSConstanciaInscripcion.new(env: :development, cuit: "20123456789", key: key, cert: cert)

ws.get_persona("20123456789")
ws.dummy
```

## Ventanilla Electrónica

```ruby
ws = Arca::VEConsumer.new(env: :development, cuit: "20123456789", key: key, cert: cert)

r = ws.consultar_comunicaciones(estado: 1, pagina: 1)
r[:items]         # las comunicaciones de esta página
r[:total_paginas]

com = ws.consumir_comunicacion(12061068, incluir_adjuntos: true)
com[:estado]                 # => "2", leída
com[:adjuntos][0][:filename] # => "informe.pdf"
com[:adjuntos][0][:content]  # el archivo, como String binario
```

`consumir_comunicacion` marca la comunicación como leída. Sin `incluir_adjuntos: true` no baja el contenido de los adjuntos, y `:adjuntos` es `[]` si no tiene ninguno.

`consultar_comunicaciones` filtra por `estado`, `fecha_desde`, `fecha_hasta`, `comunicacion_id_desde`, `comunicacion_id_hasta`, `tiene_adjunto`, `sistema_publicador_id`, `referencia1` y `referencia2`, y pagina con `pagina` y `resultados_por_pagina`.

```ruby
ws.consultar_sistemas_publicadores
ws.consultar_sistemas_publicadores(id_sistema_publicador: 88)
ws.consultar_estados # 1 = no leída, 2 = leída
```

## Declaraciones aduaneras

```ruby
ws = Arca::WConsDeclaracion.new(env: :development, cuit: "20123456789", key: key, cert: cert)

ws.detallada_lista_declaraciones(fecha_oficializacion_desde: Date.new(2025, 1, 1))
ws.detallada_lista_declaraciones(identificador_declaracion: "19093SIMI000434X")
ws.detallada_estado("19093SIMI000434X")
ws.dummy
```

`detallada_lista_declaraciones` también acepta `fecha_oficializacion_hasta`. `tipo_agente` y `rol` son `"IMEX"` por defecto y se cambian en el constructor.

## Errores

```ruby
begin
  ws.consultar_comprobante(pto_vta: 2, cbte_tipo: 1, cbte_nro: 1)
rescue Arca::ResponseError => e
  e.errors        # => [{ code: "...", msg: "..." }]
  e.code?(10016)  # => true si alguno tiene ese código
rescue Arca::NetworkError => e
  e.retriable?    # => true si no llegó a conectar
end
```

- **Arca::ResponseError** - ARCA respondió con errores; `errors` los trae con `:code` y `:msg`
- **Arca::ServerError** - un SOAP fault, un error HTTP o una respuesta que no se pudo leer
- **Arca::NetworkError** - un timeout; `retriable?` es `true` cuando falló la conexión, antes de mandar nada

Las tres heredan de `Arca::Error`.

## Cliente de bajo nivel

```ruby
client = Arca::Client.new(wsdl: "https://ejemplo.afip.gov.ar/ws/MiServicio?wsdl")

client.operations # => las operaciones del WSDL
client.request(:mi_operacion, { clave: "valor" })
```

`Arca::Client` es el cliente SOAP sobre el que se construyen todos los servicios. Usalo para un WSDL que esta gema todavía no cubre. Recibe cualquier [opción de Savon](https://www.savonrb.com/version2/globals.html), sumada a estos valores por defecto que ARCA necesita:

| Opción | Por defecto | Para qué |
|--------|-------------|----------|
| `soap_version` | `2` | SOAP 1.2 |
| `ssl_version` | `:TLSv1_2` | TLS 1.2 |
| `ssl_ciphers` | `"DEFAULT:!DH:!DHE"` | Excluye los cifrados DH, cuya clave en los servidores de ARCA es más chica de lo que OpenSSL acepta, y evita el error "dh key too small" |
| `ssl_ca_cert_file` | `OpenSSL::X509::DEFAULT_CERT_FILE` | Las autoridades certificantes del sistema |

`request(action, body = nil)` devuelve la respuesta de Savon y traduce las fallas a `Arca::ServerError` y `Arca::NetworkError`, como los servicios. El token y el sign los agregás vos al `body`, con `Arca::WSAA#auth`.

## Tests

```sh
bundle install
bundle exec rake test
```

Los tests usan los WSDL y las respuestas de `test/fixtures` y no salen a la red.

## Historia

Empezó como un fork de [eeng/afipws](https://github.com/eeng/afipws). Mirá el [changelog](CHANGELOG.md).

## Contribuir

Cualquier ayuda para mejorar el proyecto es bienvenida:

- [Reportá bugs](https://github.com/simonleva/arca.rb/issues)
- Corregí bugs y mandá pull requests
- Escribí, aclará o corregí la documentación
- Sugerí o agregá servicios nuevos

Para levantar el proyecto y publicar una versión, mirá [CONTRIBUTING.md](CONTRIBUTING.md).

## Agradecimientos

A [eeng](https://github.com/eeng), autor de [afipws](https://github.com/eeng/afipws), la gema de la que salió esta.

## Licencia

MIT. Mirá [LICENSE](LICENSE).
