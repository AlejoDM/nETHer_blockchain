Contrato N°1 - GestionReservas.sol
// SPDX-License-Identifier: MIT
pragma solidity 0.8.34;




/// @dev Interfaz para interactuar de forma externa con el contrato de custodia financiera[cite: 113].
interface IGarantiaContrato {
    function retenerFondos(uint256 _idReserva, address payable _propietario, address _inquilino) external payable; // [cite: 114, 115, 116]
    function liberarFondos(uint256 _idReserva) external; // [cite: 118]
    function reembolsarFondos(uint256 _idReserva) external; // [cite: 119]
}




/// @title Contrato de Gestión de Alquileres Temporales para nETHER [cite: 120]
/// @author Martín Beccereca & Alejo De Miguel [cite: 121]
/// @notice Administra el alta de inmuebles, solicitudes de reserva, check-outs y estados de disputa[cite: 122].
contract GestionReservas {
   
    address public immutable administrador; // [cite: 124]
    IGarantiaContrato public bovedaGarantia; // [cite: 124]
    uint256 public contadorReservas; // [cite: 125]




    enum EstadoReserva { Inexistente, Disponible, Reservada, Finalizada, EnDisputa } // [cite: 126]




    struct Reserva {
        address payable propietario;
        address inquilino;
        uint256 precioPorNoche;
        EstadoReserva estado;
    } // [cite: 127, 129, 130, 131, 132]




    // Mapeo de idReserva => Estructura de la Reserva [cite: 133, 134]
    mapping(uint256 => Reserva) public historialReservas;




    // Custom Errors para optimizar consumo de gas [cite: 135]
    error SoloAdministrador();
    error SoloInquilinoAutorizado();
    error EstadoInvalido();
    error FondosInsuficientes();
    error BovedaNoConfigurada();




    // Eventos del ciclo de negocio [cite: 141]
    event InmuebleRegistrado(uint256 indexed idReserva, address indexed propietario, uint256 precio); // [cite: 147]
    event EstadiaReservada(uint256 indexed idReserva, address indexed inquilino, uint256 montoTotal); // [cite: 148]
    event EstadiaFinalizada(uint256 indexed idReserva); // [cite: 149]
    event DisputaAbierta(uint256 indexed idReserva); // [cite: 150]
    event DisputaResuelta(uint256 indexed idReserva, string resolucion); // [cite: 151]




    modifier soloAdmin() {
        if (msg.sender != administrador) revert SoloAdministrador(); // [cite: 152, 153]
        _;
    }




    constructor() {
        administrador = msg.sender; // [cite: 155, 156]
    }




    /// @notice Enlaza la dirección del contrato de custodia financiera de forma definitiva[cite: 158, 159].
    /// @param _boveda Dirección implementada del contrato GarantiaContrato[cite: 160, 161].
    function setBoveda(address _boveda) external soloAdmin {
        bovedaGarantia = IGarantiaContrato(_boveda); // [cite: 162, 164]
    }




    /// @notice Permite dar de alta un inmueble estableciendo su propietario y precio base[cite: 165].
    function registrarInmueble(uint256 _idInmueble, address payable _propietario, uint256 _precio) external {
        if (historialReservas[_idInmueble].estado != EstadoReserva.Inexistente) revert EstadoInvalido(); // [cite: 169, 170]
       
        historialReservas[_idInmueble] = Reserva({
            propietario: _propietario,
            inquilino: address(0),
            precioPorNoche: _precio,
            estado: EstadoReserva.Disponible
        }); // [cite: 171, 172, 173, 174]




        emit InmuebleRegistrado(_idInmueble, _propietario, _precio); // [cite: 181]
    }




    /// @notice Ejecuta el alquiler exigiendo de forma obligatoria el precio base + fianza del 5%.
    /// @dev Cumple con el Patrón CEI (Checks-Effects-Interactions)[cite: 182, 183].
    function alquilar(uint256 _idInmueble) external payable {
        if (address(bovedaGarantia) == address(0)) revert BovedaNoConfigurada(); // [cite: 185, 186]
        Reserva storage unaReserva = historialReservas[_idInmueble]; // [cite: 187]
       
        // 1. CHECKS (Validaciones) [cite: 188]
        if (unaReserva.estado != EstadoReserva.Disponible) revert EstadoInvalido(); // [cite: 189, 190]
       
        // El contrato calcula on-chain el 5% extra por encima de la tarifa base
        uint256 fianzaRequerida = (unaReserva.precioPorNoche * 5) / 100;
        uint256 costoTotalConGarantia = unaReserva.precioPorNoche + fianzaRequerida;
       
        if (msg.value < costoTotalConGarantia) revert FondosInsuficientes(); // [cite: 191, 192]




        // 2. EFFECTS (Cambios de estado internos locales) [cite: 193]
        contadorReservas++; // [cite: 194]
        unaReserva.inquilino = msg.sender; // [cite: 195]
        unaReserva.estado = EstadoReserva.Reservada; // [cite: 196, 197]




        emit EstadiaReservada(_idInmueble, msg.sender, msg.value); // [cite: 198]




        // 3. INTERACTIONS (Se delega la custodia completa de los fondos a la bóveda) [cite: 199]
        bovedaGarantia.retenerFondos{value: msg.value}(_idInmueble, unaReserva.propietario, msg.sender); // [cite: 199, 200]
    }




    /// @notice Confirma el check-out de la estadía y gatilla la liquidación comercial[cite: 202].
    function confirmarCheckOut(uint256 _idInmueble) external {
        Reserva storage unaReserva = historialReservas[_idInmueble]; // [cite: 204, 205]
       
        if (msg.sender != unaReserva.inquilino) revert SoloInquilinoAutorizado(); // [cite: 206, 207]
        if (unaReserva.estado != EstadoReserva.Reservada) revert EstadoInvalido(); // [cite: 208, 209]




        unaReserva.estado = EstadoReserva.Finalizada; // [cite: 215, 216]
        emit EstadiaFinalizada(_idInmueble); // [cite: 217]




        // Interacción cross-contract para pagarle al dueño y devolver la fianza al cliente
        bovedaGarantia.liberarFondos(_idInmueble); // [cite: 218, 219]
    }




    /// @notice Permite al inquilino congelar el dinero on-chain abriendo una disputa si hubo fraude[cite: 220].
    function abrirDisputa(uint256 _idInmueble) external {
        Reserva storage unaReserva = historialReservas[_idInmueble]; // [cite: 221, 222]
       
        if (msg.sender != unaReserva.inquilino) revert SoloInquilinoAutorizado(); // [cite: 223, 224]
        if (unaReserva.estado != EstadoReserva.Reservada) revert EstadoInvalido(); // [cite: 225, 226]




        unaReserva.estado = EstadoReserva.EnDisputa; // [cite: 227, 228]
        emit DisputaAbierta(_idInmueble); // [cite: 229]
    }




    /// @notice Resolución arbitral definitiva de una disputa activa llevada a cabo por el administrador[cite: 231].
    function resolverDisputa(uint256 _idInmueble, bool _devolverAlInquilino) external soloAdmin {
        Reserva storage unaReserva = historialReservas[_idInmueble]; // [cite: 235, 236]
        if (unaReserva.estado != EstadoReserva.EnDisputa) revert EstadoInvalido(); // [cite: 236, 237]




        if (_devolverAlInquilino) {
            unaReserva.estado = EstadoReserva.Disponible; // [cite: 239, 241]
            emit DisputaResuelta(_idInmueble, "Reembolso Total Ejecutado"); // [cite: 242]
            bovedaGarantia.reembolsarFondos(_idInmueble); // [cite: 242]
        } else {
            unaReserva.estado = EstadoReserva.Finalizada; // [cite: 246]
            emit DisputaResuelta(_idInmueble, "Pago a Propietario Forzado"); // [cite: 246]
            bovedaGarantia.liberarFondos(_idInmueble); // [cite: 247]
        }
    }
}

Contrato N° 2 - GarantiaContrato.sol
// SPDX-License-Identifier: MIT
pragma solidity 0.8.34;




/// @title Contrato de Bóveda y Custodia de Fondos (Escrow Vault) para nETHER
/// @author Martín Beccereca & Alejo De Miguel
/// @notice Almacena el Ether transaccionado, distribuyendo el alquiler y reintegrando la fianza del 5%.
contract GarantiaContrato {




    address public immutable contratoComercial; // Dirección del gestor de reservas




    struct Custodia {
        address payable propietario;
        address payable inquilino;
        uint256 montoTotal; // Alquiler + 5% de depósito
        bool liquidado;
    }




    // Mapeo indexado por el ID del inmueble/reserva
    mapping(uint256 => Custodia) public mapaCustodias;




    // Custom Errors para optimizar gas
    error SoloContratoComercialAutorizado();
    error FondosYaLiquidados();
    error TransferenciaInvalida();




    // Eventos de la bóveda
    event FondosBloqueados(uint256 indexed idReserva, address indexed inquilino, uint256 monto);
    event FondosEnviadosAPropietario(uint256 indexed idReserva, address indexed propietario, uint256 monto);
    event GarantiaDevueltaAInquilino(uint256 indexed idReserva, address indexed inquilino, uint256 monto);
    event FondosDevueltosAInquilino(uint256 indexed idReserva, address indexed inquilino, uint256 monto);




    // Restringe el acceso únicamente al contrato comercial legítimo
    modifier soloComercial() {
        if (msg.sender != contratoComercial) revert SoloContratoComercialAutorizado();
        _;
    }




    /// @notice Inicializa el resguardo enlazándolo al contrato comercial emisor.
    /// @param _contratoComercial Dirección del contrato GestionReservas.
    constructor(address _contratoComercial) {
        contratoComercial = _contratoComercial; //
    }




    /// @notice Retiene el Ether enviado por el inquilino asociándolo a la reserva.
    function retenerFondos(uint256 _idReserva, address payable _propietario, address _inquilino) external payable soloComercial {
        mapaCustodias[_idReserva] = Custodia({
            propietario: _propietario,
            inquilino: payable(_inquilino),
            montoTotal: msg.value,
            liquidado: false
        }); //




        emit FondosBloqueados(_idReserva, _inquilino, msg.value); //
    }




    /// @notice Divide los fondos: envía el 100% del alquiler al dueño y devuelve el 5% de fianza al inquilino.
    /// @dev Cumple con el patrón CEI (Checks-Effects-Interactions).
    function liberarFondos(uint256 _idReserva) external soloComercial {
        Custodia storage custodia = mapaCustodias[_idReserva]; //
       
        // 1. CHECKS
        if (custodia.liquidado) revert FondosYaLiquidados(); //




        // 2. EFFECTS
        custodia.liquidado = true; //
        uint256 totalAcumulado = custodia.montoTotal;
       
        // Desglose matemático: totalAcumulado representa el 105% del valor base
        uint256 montoAlquiler = (totalAcumulado * 100) / 105;
        uint256 montoGarantia = totalAcumulado - montoAlquiler;




        // 3. INTERACTIONS (Llamadas externas seguras con .call)
       
        // Transferencia del valor neto del alquiler al propietario
        (bool exitoPropietario, ) = custodia.propietario.call{value: montoAlquiler}(""); //
        if (!exitoPropietario) revert TransferenciaInvalida(); //
        emit FondosEnviadosAPropietario(_idReserva, custodia.propietario, montoAlquiler);




        // Devolución automática del 5% de depósito de fianza al inquilino
        (bool exitoInquilino, ) = custodia.inquilino.call{value: montoGarantia}("");
        if (!exitoInquilino) revert TransferenciaInvalida();
        emit GarantiaDevueltaAInquilino(_idReserva, custodia.inquilino, montoGarantia);
    }




    /// @notice En caso de disputa a favor del cliente, reembolsa el 100% depositado (Alquiler + Fianza).
    /// @dev Cumple con el patrón CEI (Checks-Effects-Interactions).
    function reembolsarFondos(uint256 _idReserva) external soloComercial {
        Custodia storage custodia = mapaCustodias[_idReserva]; //
       
        // 1. CHECKS
        if (custodia.liquidado) revert FondosYaLiquidados(); //




        // 2. EFFECTS
        custodia.liquidado = true; //
        uint256 montoReembolso = custodia.montoTotal;




        // 3. INTERACTIONS
        (bool exito, ) = custodia.inquilino.call{value: montoReembolso}(""); //
        if (!exito) revert TransferenciaInvalida(); //




        emit FondosDevueltosAInquilino(_idReserva, custodia.inquilino, montoReembolso); //
    }
}
