# evidencia-programacion
programa de la evidencia


import datetime  # Importamos la librería datetime para manejar fechas (día, mes, año)

# ==========================
#   ESTRUCTURAS DE DATOS EN MEMORIA
# ==========================
# Vamos a guardar todo en RAM, como pide el profe (no se usan archivos).

clientes = {}      # Aquí guardamos clientes. Ejemplo: { 1: {"nombre": "Ana", "apellidos": "Pérez"} }
salas = {}         # Aquí guardamos salas. Ejemplo: { 1: {"nombre": "Sala Azul", "cupo": 10} }
reservas = {}      # Aquí guardamos reservaciones. Ejemplo:
                   # { "R001": {"cliente": 1, "sala": 1, "fecha": date(2025,10,20), "turno": "M", "evento": "Reunión"} }

# Contadores para generar claves únicas de forma automática
sig_cliente_id = 1   # Clave incremental para clientes (1, 2, 3, ...)
sig_sala_id = 1      # Clave incremental para salas
sig_reserva_num = 1  # Folio incremental para reservaciones (R001, R002, ...)

# Turnos disponibles en el negocio
TURNOS = {"M": "Matutino", "V": "Vespertino", "N": "Nocturno"}


# ==========================
#   FUNCIONES AUXILIARES
# ==========================

def generar_folio_reserva(n):
    # Genera un folio con el formato R001, R002, etc.
    return f"R{n:03d}"

def hoy():
    # Devuelve la fecha actual del sistema (la de la computadora)
    return datetime.date.today()

def leer_fecha_con_cancel(mensaje):
    # Pide al usuario una fecha en formato AAAA-MM-DD
    # Si el usuario escribe 'C', se cancela y devuelve None
    # Si el formato de fecha está mal, devuelve "INVALIDA"
    crudo = input(mensaje).strip()
    if crudo.upper() == "C":
        return None
    try:
        return datetime.datetime.strptime(crudo, "%Y-%m-%d").date()
    except ValueError:
        return "INVALIDA"

def fecha_reservable(f):
    # Verifica que la fecha sea válida y al menos 2 días después de hoy
    return f is not None and f != "INVALIDA" and f >= hoy() + datetime.timedelta(days=2)

def clientes_ordenados():
    # Devuelve la lista de clientes ordenados por Apellidos y luego por Nombre
    def clave_orden(kv):
        return kv[1]["apellidos"], kv[1]["nombre"]
    return sorted(clientes.items(), key=clave_orden)

def turnos_ocupados(fecha, sala_id):
    # Devuelve los turnos que ya están ocupados en una sala para una fecha
    ocupados = []
    for r in reservas.values():
        if r["fecha"] == fecha and r["sala"] == sala_id:
            ocupados.append(r["turno"])
    return ocupados

def turnos_disponibles(fecha, sala_id):
    # Devuelve los turnos que todavía están libres en una sala y fecha
    ocup = set(turnos_ocupados(fecha, sala_id))
    return {k: v for k, v in TURNOS.items() if k not in ocup}

def salas_con_disponibilidad_en(fecha):
    # Devuelve todas las salas que aún tienen turnos disponibles en la fecha
    disponibles = {}
    for sala_id in salas:
        libres = set(turnos_disponibles(fecha, sala_id).keys())
        if libres:
            disponibles[sala_id] = libres
    return disponibles

def pedir_nombre_evento():
    # Pide un nombre de evento obligatorio (no puede estar vacío)
    while True:
        nombre = input("Nombre del evento: ").strip()
        if nombre:
            return nombre
        print("El nombre no puede estar vacío.")


# ==========================
#   FUNCIONES PARA ELEGIR OPCIONES
# ==========================

def elegir_cliente():
    # Muestra los clientes y permite elegir uno
    if not clientes:
        print("No hay clientes registrados.")
        return None

    while True:
        print("\nClientes registrados:")
        for cid, datos in clientes_ordenados():
            print(f"  {cid} - {datos['apellidos']}, {datos['nombre']}")
        entrada = input("Ingrese la clave del cliente (o 'C' para cancelar): ").strip()

        if entrada.upper() == "C":
            return None  # Cancelar

        try:
            clave = int(entrada)  # Intentamos convertir la entrada a número
        except ValueError:
            print("Debe ingresar un número de cliente.")
            continue

        if clave in clientes:
            return clave
        print("Cliente no encontrado, se vuelve a mostrar la lista.")

def elegir_fecha_reserva():
    # Pide y valida una fecha para la reservación
    while True:
        f = leer_fecha_con_cancel("Fecha del evento (AAAA-MM-DD) o 'C' para cancelar: ")
        if f is None:
            return None
        if f == "INVALIDA":
            print("Formato inválido, use AAAA-MM-DD.")
            continue
        if fecha_reservable(f):
            return f
        print("La fecha debe ser al menos dos días después de hoy.")

def elegir_sala_para(fecha):
    # Muestra salas disponibles y permite elegir una
    if not salas:
        print("No hay salas registradas.")
        return None

    while True:
        mapa = salas_con_disponibilidad_en(fecha)
        if not mapa:
            print("No hay salas con turnos disponibles en esa fecha.")
            return None

        print("\nSalas disponibles:")
        for sala_id, libres in sorted(mapa.items()):
            datos = salas[sala_id]
            codigos = ", ".join(sorted(libres))  # Ejemplo: M, V
            print(f"  {sala_id} - {datos['nombre']} (Cupo: {datos['cupo']}) | Turnos libres: {codigos}")

        entrada = input("Ingrese la clave de sala (o 'C' para cancelar): ").strip()
        if entrada.upper() == "C":
            return None
        try:
            sid = int(entrada)
        except ValueError:
            print("Debe ingresar un número de sala.")
            continue
        if sid in mapa:
            return sid
        print("Sala no válida, se vuelve a mostrar la lista.")

def elegir_turno_para(fecha, sala_id):
    # Muestra turnos disponibles y pide uno válido
    while True:
        libres = turnos_disponibles(fecha, sala_id)
        if not libres:
            print("No hay turnos libres en esa sala/fecha.")
            return None
        print("Turnos disponibles:", ", ".join(f"{k}={v}" for k, v in libres.items()))
        t = input("Elija turno (M/V/N) o 'C' para cancelar: ").strip().upper()
        if t == "C":
            return None
        if t in libres:
            return t
        print("Turno inválido, intente de nuevo.")


# ==========================
#   FUNCIONES PRINCIPALES DEL MENÚ
# ==========================

def registrar_cliente():
    # Registrar un cliente con clave automática
    global sig_cliente_id
    nombre = input("Nombre: ").strip()
    apellidos = input("Apellidos: ").strip()
    if not nombre or not apellidos:
        print("Datos inválidos.")
        return
    clientes[sig_cliente_id] = {"nombre": nombre, "apellidos": apellidos}
    print(f"Cliente registrado con clave {sig_cliente_id}")
    sig_cliente_id += 1

def registrar_sala():
    # Registrar una sala con nombre y cupo
    global sig_sala_id
    nombre = input("Nombre de la sala: ").strip()
    try:
        cupo = int(input("Cupo (entero > 0): ").strip())
    except ValueError:
        print("El cupo debe ser un número entero.")
        return
    if not nombre or cupo <= 0:
        print("Datos inválidos.")
        return
    salas[sig_sala_id] = {"nombre": nombre, "cupo": cupo}
    print(f"Sala registrada con clave {sig_sala_id}")
    sig_sala_id += 1

def registrar_reservacion():
    # Registrar una reservación paso a paso
    global sig_reserva_num

    if not clientes or not salas:
        print("Debe haber al menos un cliente y una sala antes de reservar.")
        return

    cliente_id = elegir_cliente()
    if cliente_id is None:
        print("Operación cancelada.")
        return

    fecha = elegir_fecha_reserva()
    if fecha is None:
        print("Operación cancelada.")
        return

    sala_id = elegir_sala_para(fecha)
    if sala_id is None:
        print("Operación cancelada.")
        return

    turno = elegir_turno_para(fecha, sala_id)
    if turno is None:
        print("Operación cancelada.")
        return

    # Verificar que no exista otra reservación igual
    for r in reservas.values():
        if r["fecha"] == fecha and r["sala"] == sala_id and r["turno"] == turno:
            print("Esa sala ya está reservada en ese turno y fecha.")
            return

    evento = pedir_nombre_evento()

    folio = generar_folio_reserva(sig_reserva_num)
    reservas[folio] = {
        "cliente": cliente_id,
        "sala": sala_id,
        "fecha": fecha,
        "turno": turno,
        "evento": evento,
    }
    sig_reserva_num += 1
    print(f"Reservación registrada con folio {folio}")

def reporte_tabular_por_fecha():
    # Muestra reservaciones en formato tabular (estilo Figura 1)
    f = leer_fecha_con_cancel("Fecha a consultar (AAAA-MM-DD) o 'C' para cancelar: ")
    if f is None:
        print("Operación cancelada.")
        return
    if f == "INVALIDA":
        print("Fecha inválida.")
        return

    filas = []
    for r in reservas.values():
        if r["fecha"] == f:
            cli = clientes[r["cliente"]]
            cliente_txt = f"{cli['apellidos']}, {cli['nombre']}"
            turno_txt = TURNOS[r["turno"]].upper()
            filas.append((r["sala"], cliente_txt, r["evento"], turno_txt))

    ancho = 74
    print("*" * ancho)
    titulo = f"REPORTE DE RESERVACIONES PARA EL DÍA {f}"
    print("**" + f"{titulo:^{ancho-4}}" + "**")
    print("*" * ancho)
    print("{:<6} {:<18} {:<38} {:<11}".format("SALA", "CLIENTE", "EVENTO", "TURNO"))
    print("*" * ancho)

    if not filas:
        print("No hay reservaciones en esa fecha.")
    else:
        for sala_id, cliente_txt, evento, turno_txt in filas:
            print("{:<6} {:<18} {:<38} {:<11}".format(
                sala_id, cliente_txt[:18], evento[:38], turno_txt
            ))

    print("******************** FIN DEL REPORTE ********************")

def editar_nombre_evento():
    # Edita el nombre de un evento en un rango de fechas
    inicio = leer_fecha_con_cancel("Fecha inicio (AAAA-MM-DD) o 'C' para cancelar: ")
    if inicio is None or inicio == "INVALIDA":
        print("Operación cancelada o inválida.")
        return

    fin = leer_fecha_con_cancel("Fecha fin (AAAA-MM-DD) o 'C' para cancelar: ")
    if fin is None or fin == "INVALIDA" or fin < inicio:
        print("Operación cancelada o rango inválido.")
        return

    en_rango = {folio: r for folio, r in reservas.items() if inicio <= r["fecha"] <= fin}
    if not en_rango:
        print("No hay eventos en ese rango.")
        return

    while True:
        print("\nEventos en el rango:")
        print("{:<6} {:<12} {}".format("FOLIO", "FECHA", "EVENTO"))
        print("-" * 40)
        for folio, r in sorted(en_rango.items(), key=lambda kv: (kv[1]["fecha"], kv[0])):
            print("{:<6} {:<12} {}".format(folio, str(r["fecha"]), r["evento"]))

        eleccion = input("Ingrese folio a modificar (o 'C' para cancelar): ").strip().upper()
        if eleccion == "C":
            print("Operación cancelada.")
            return
        if eleccion in en_rango:
            break
        print("Folio no válido, se vuelve a mostrar la lista.")

    nuevo = pedir_nombre_evento()
    reservas[eleccion]["evento"] = nuevo
    print("Evento actualizado.")


# ==========================
#   MENÚ PRINCIPAL
# ==========================

def menu():
    while True:
        print("\n--- Menú ---")
        print("1. Registrar reservación de sala")
        print("2. Editar nombre de evento")
        print("3. Consultar reservaciones por fecha")
        print("4. Registrar nuevo cliente")
        print("5. Registrar nueva sala")
        print("6. Salir")

        try:
            op = int(input("Elija una opción: ").strip())
        except ValueError:
            print("Debe ingresar un número de opción.")
            continue

        if op == 1:
            registrar_reservacion()
        elif op == 2:
            editar_nombre_evento()
        elif op == 3:
            reporte_tabular_por_fecha()
        elif op == 4:
            registrar_cliente()
        elif op == 5:
            registrar_sala()
        elif op == 6:
            print("Adiós.")
            break
        else:
            print("Opción no válida.")


# ==========================
#   EJECUCIÓN DEL PROGRAMA
# ==========================

if __name__ == "__main__":
    # Inicia el programa mostrando el menú
    menu()
