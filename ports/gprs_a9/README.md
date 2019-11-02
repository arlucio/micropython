Your support is important for this port: please consider donating.

[![paypal](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=2RZCNXCUCP4YG&source=url)

[![Build Status](https://travis-ci.org/pulkin/micropython.svg?branch=master)](https://travis-ci.org/pulkin/micropython)
[![Build Status](https://dev.azure.com/gpulkin/micropython/_apis/build/status/pulkin.micropython?branchName=master)](https://dev.azure.com/gpulkin/micropython/_build/latest?definitionId=1&branchName=master)

## Build

The firmware image is automatically built by Azure pipelines: [download](https://github.com/pulkin/micropython/releases/tag/latest-build).
Follow these steps to build from sources:

1. Install dependencies (Ubuntu example from `.travis.yml`):
   ```bash
   sudo apt-get install build-essential gcc-multilib g++-multilib libzip-dev zlib1g lib32z1
   ```
2. Clone this (install `sudo apt install git` if necessary)
   **Warning**: large download size
   ```bash
   git clone git@github.com:pulkin/micropython.git --recursive
   ```
   or (smaller download size):
   ```bash
   git clone git@github.com:pulkin/micropython.git
   git submodule update --init --recursive lib/axtls lib/GPRS_C_SDK lib/csdtk42-linux
   ```
3. Make
   ```bash
   cd micropython
   make -C mpy-cross
   cd ports/gprs_a9
   make
   ```

## Burn

Follow vendor [documentation](https://ai-thinker-open.github.io/GPRS_C_SDK_DOC/en/c-sdk/installation_linux.html)
(`cooltools` is available under `lib/csdtk42-linux/cooltools/`).

## Connect

Use [pyserial](https://github.com/pyserial) or any other terminal.

```bash
miniterm.py /dev/ttyUSB1 115200 --raw
```

## Upload scripts

Use [ampy](https://github.com/pycampers/ampy).

```bash
ampy --port /dev/ttyUSB1 put frozentest.py 
```

## Run scripts

```python
>>> help()
>>> import frozentest
```

## Functionality

- [x] GPIO: `machine.Pin`
- [ ] ADC: `machine.ADC`
- [ ] PWM: `machine.PWM`
- [ ] UART: `machine.UART` (software UART?)
- [x] Cellular misc (IMEI, ICCID, ...): `cellular`
- [x] GPS: `gps`
- [x] I2C: `i2c`
- [ ] SPI: `machine.SPI`
- [x] time: `utime`
- [x] File system
- [x] GPRS, DNS: `cellular`, `socket`, `ssl`
- [x] Power: `machine`
- [ ] Calls: `cellular`
- [x] SMS: `cellular.SMS`

## API

1. [`cellular`](#cellular): SMS, calls, connectivity
2. [`usocket`](#usocket): sockets over GPRS
3. [`ssl`](#ussl): SSL over sockets
4. [`gps`](#gps): everything related to GPS and assisted positioning
5. [`machine`](#machine): hardware and power control
6. [`i2c`](#i2c): i2c implementation
8. [Notes](#Notes)

### `cellular`

Provides cellular functionality.
As usual, the original API does not give access to radio-level and low-level functionality such as controlling the registration on the cellular network: these are performed in the background automatically.
The purpose of this module is to have an access to high-level networking (SMS, GPRS, calls) as well as to read the status of various components of cellular networking.

#### Classes ####

* `SMS(phone_number, message)`

  A class for handling SMS.

  **Attrs**:

    * phone_number (str): phone number (sender or destination);
    * message (str): message contents;
    * status (int): an integer with status bits;
    * inbox (bool): incoming message if `True`, outgoing message if `False` or unknown status if `None`;
    * unread (bool): unread message if `True`, previously read message if `False` or unknown status if `None`;
    * sent (bool): sent message if `True`, not sent message if `False` or unknown status if `None`;
    * `send()`
    
      Sends the message.

      **Raises**: `CelularError` if not registered on the network or failed to set up/send SMS.

    * `withdraw()`

      Withdraws SMS from SIM storage. Resets status and index of this object to zero.

      **Raises**: `CellularError` if failed to withdraw.

    * `poll()` [staticmethod]

      Polls SMS.

      **Returns**: the number of new SMS received.

    * `list()` [staticmethod]

      Retrieves SMS from the SIM card.
      
      **Returns**: a list of SMS messages.

      **Raises**: `CellularError` if not registered on the network. This is because network registration process interfers with most other SIM-related operations.

#### Exception classes ####

* `CellularError(message)`

#### Methods ####

##### Status and information #####

* `get_imei()`

  Retrieves the International Mobile Equipment Identity (IMEI) number.

  **Returns**: a string with IMEI number.

* `get_iccid()`

  Retrieves the Integrated Circuit Card ID (ICCID) number of the inserted SIM card.

  **Returns**: a string with ICCID number.

  **Raises**: `CellularError` if no ICCID number can be retrieved.

* `get_imsi()`

  Retrieves the International Mobile Subscriber Identity (IMSI) number of the inserted SIM card.

  **Returns**: a string with IMSI number.

  **Raises**: `CellularError` if no IMSI number can be retrieved.

* `network_status_changed()`

  Checks whether the network status was changed since the last check.

  **Returns**: `True` if it was changed.

* `get_network_status()`

  Retrieves the network status as an integer.

  **Returns**: `int` representing cellular network status.

  **TODO**: Provide bit-wise specs.

* `poll_network_exception()`

  Retrieves the network exception and raises it, if any.

  **Raises**: One of `CellularError`s occurred during the operation in the background.

* `is_sim_present()`

  Checks whether the SIM card is present and ICCID can be retrieved.

  **Returns**: True if SIM card is present.

* `is_network_registered()`

  Checks whether registered on the cellular network.

  **Returns**: True if registered.

* `is_roaming()`

  Checks whether registered on the roaming network.

  **Returns**: True if roaming.

  **Raises** `CellularError` if not registered on the network.

* `get_signal_quality()`

  Retrieves signal quality.

  **Returns**: Two integers, the signal quality (0-31) and RXQUAL. These are replaced by `None` if no signal quality information is available.
  **Note**: The RXQUAL output is always `None`. Its meaning is unknown.

* `flight_mode(flag)`

  Turn the flight mode on or off. Retrieve the flight mode status.

  **Args**:

    * flag (bool): set or unset the flight mode;

  **Returns**: True if in flight mode.

* `reset()`

  Resets network settings to defaults. Disconnects GPRS.

  **Raises**: `CellularError` if reset failed at any stage.

##### GPRS #####

* `gprs(apn, user=None, pass=None)`

  Activates, deactivates or retrieves the status of GPRS.

  Activate: `cellular.gprs('apn', 'user', 'pass')`

  Deactivate: `cellular.gprs(False)`

  Status: `cellular.gprs()`

  **Args**:

    * apn (str, bool): access point name (APN);
    * user (str): username;
    * pass (str): password;

  **Returns**: True if GPRS is operating.

  **Raises**: `CellularError` if not registered on the network or the (de)activation process failed at any stage.

  **Note**: there is no way to check whether the credentials supplied are valid.

### `usocket` ###

*Alias: `socket`*

TCP/IP stack over GPRS based on lwIP.
See [micropython docs](https://docs.micropython.org/en/latest/library/usocket.html) for details.

#### Constants ####

`AF_INET`, `AF_INET6`, `SOCK_STREAM`, `SOCK_DGRAM`, `SOCK_RAW`, `IPPROTO_TCP`, `IPPROTO_UDP`, `IPPROTO_IP`

#### Classes ####

* `socket(af, type, proto)`

    * `close()`
    * `bind(address)` *not implemented*
    * `listen([backlog])` *not implemented*
    * `accept()` *not implemented*
    * `connect(address)`
    * `send(bytes)`
    * `sendall(bytes)`
    * `recv(bufsize)`
    * `sendto(bytes, address)`
    * `recvfrom(bufsize)`
    * `setsockopt(level, optname, value)` *not implemented*
    * `settimeout(value)` *not implemented*
    * `setblocking(flag)` *not implemented*
    * `makefile(mode, buffering)`
    * `read([size])`
    * `readinto(buf[, nbytes])`
    * `readline()`
    * `write(buf)`

#### Methods ####

* `get_local_ip()`

  Retrieves the local IP address.

  **Returns**: The local IP address as a string.

  **Raises**: `NetworkError` if no address was assigned.

* `getaddrinfo(host, port, af=AF_INET, type=SOCK_STREAM, proto=IPPROTO_TCP, flags=0)`

  Translates host/port into arguments to socket constructor.

  **Args**:

    * host (str): host name;
    * port (int): port number;
    * af (int): address family: `AF_INET` or `AF_INET6`;
    * type: (int): future socket type: `SOCK_STREAM` or `SOCK_DGRAM`;
    * proto (int): future protocol: `IPPROTO_TCP` or `IPPROTO_UDP`;
    * flag (int): additional socket flags;

  **Returns**: a tuple with arguments.

* `inet_ntop(af, bin_addr)`

  *Not implemented*

  Converts a binary address into textual representation.

  **Args**:
  
    * af (int): address family: `AF_INET` or `AF_INET6`;
    * bin_addr (bytearray): binary address;

  **Returns**: a string with the address.

* `inet_pton(af, txt_addr)`

  *Not implemented*

  Converts a text address into binary representation.

  **Args**:

    * af (int): address family: `AF_INET` or `AF_INET6`;
    * txt_addr (str): address as text;

  **Returns**: a bytearray address.

* `get_num_open()`

  Retrieves the number of open sockets.

  **Returns**: The number of open sockets.

### `ussl` ###

*Alias: `ssl`*

Provides SSL over GPRS sockets (axtls).

#### Classes ####

* `ssl_socket(af, type, proto)`

    * `close()`
    * `read([size])`
    * `readinto(buf[, nbytes])`
    * `readline()`
    * `write(buf)`

#### Methods ####

* `wrap_socket(sock, server_side=False, keyfile=None, certfile=None, cert_reqs=CERT_NONE, ca_certs=None)`

  Takes a stream socket and returns an `SSLSocket`.

  **Args**:

    * sock (`usocket.socket`): the socket to wrap;

  **Returns**: a wrapped SSL socket.

### `gps`

Provides the GPS functionality.
This is only available in the A9G module where GPS is a separate chip connected via UART2.

#### Exception classes ####

* `GPSError(message)`

#### Methods ####

* `on()`

  Turns the GPS on. Blocks until the GPS module responds.

  **Raises**: `GPSError` if the GPS module does not respond within 10 seconds.

* `off()`

  Turns the GPS off.

* `get_firmware_version()`

  Retrieves the firmware version.

  **Returns**: the firmware version as a string.

  **Raises**: `GPSError` if the GPS module fails to respond.

* `get_location()`

  Retrieves the current GPS location.

  **Returns**: latitude and longitude in degrees.

  **Raises**: `GPSError` if the GPS module never responded or is off.

* `get_last_location()`

  Retrieves the last GPS location.

  **Returns**: latitude and longitude in degrees.

  **Raises**: `GPSError` if the GPS module never responded.

* `get_satellites()`

  Retrieves the number of satellites visible.

  **Returns**: the number of satellites tracked and the number of visible satellites.

  **Raises**: `GPSError` if the GPS module is off.

### `machine`

Provides power-related functions: power, watchdogs.

#### Methods ####

* `reset()`

  Hard-resets the module.

* `off()`

  Powers the module down.
  **Note**: By fact, hard-resets the module, at least when USB-powered.
  **TODO**: Needs further investigation.

* `idle()`

  Reduces clock rates of the module keeping functionality.

* `get_input_voltage()`

  Retrieves the input voltage and the battery percentage.

  **Returns**: two numbers: the voltage in `mV` and the estimated percentage (Lithium battery discharge controller).
  **Note**: also works when USB-powered (the second number returned is irrelevant).

* `power_on_cause()`

  Retrieves the reason for powering the module on.

  **Returns**: one of `POWER_ON_CAUSE_ALARM`, `POWER_ON_CAUSE_CHARGE`, `POWER_ON_CAUSE_EXCEPTION`, `POWER_ON_CAUSE_KEY`, `POWER_ON_CAUSE_MAX`, `POWER_ON_CAUSE_RESET`.
  **Note**: never saw anything except `POWER_ON_CAUSE_CHARGE` returned.
  **TODO**: needs further investigation.

* `watchdog_on(timeout)`

  Arms the hardware watchdog.

  **Args**:

    * timeout (int): timeout in seconds;

* `watchdog_off()`

  Disarms the hardware watchdog.

* `watchdog_reset()`

  Resets the timer on the hardware watchdog.
  
### `i2c`

Provides i2c functionality
(see [sdk docs](https://ai-thinker-open.github.io/GPRS_C_SDK_DOC/en/c-sdk/function-api/iic.html))

#### Exception classes ####

* `I2CError(message)`

#### Constants ####

I2C_DEFAULT_TIME_OUT

#### Enums ####

* `id`

  **Options**: `1`, `2`, `3`
  
  **Fallback**: `1`
  
  **Meaning**: I2C port ids (see devboard layout)
  
  
* `freq`

  **Options**: `100`, `400`
  
  **Fallback**: `100`
  
  **Meaning**: I2C SCL Frequency 100kHz, 400kHz respectively
  

#### Methods ####

**Note**: parameters in square brackets are optional

* `init(id, freq)`

  Inintializes i2c on given port id and frequency
    
  **Args**:

    * `id` (int): see enums;
    * `freq` (int): see enums;
  
  **Returns**: `None` if everything ok.

  **Raises** `I2CError` (see error message for more details)
  
  **Note**: Different ports can be initialized simultaneously

* `close(id)`

  Closes i2c on given port id
  
  **Args**:

    * `id` (int): see enums;
  
  **Returns**: `None` if everything ok.

  **Raises** `I2CError` (see error message for more details)
  
* `receive(id, slave_address, data_length, [timeout] = I2C_DEFAULT_TIME_OUT)`

  Receive data given length from slave device
  
  **Args**:

    * `id` (int): see enums;
    * `slave_address` (int, hex): slave device communication address;
    * `data_length` (int): data length needed to be received;
    * `[timeout]` (int) = 10: timeout of receiving in ms
  
  **Returns**: bytes() of given length.

  **Raises** `I2CError` (see error message for more details)
  
* `transmit(id, slave_address, data, [timeout] = I2C_DEFAULT_TIME_OUT)`

  Transmit given data to slave device
  
  **Args**:

    * `id` (int): see enums;
    * `slave_address` (int, hex): slave device communication address;
    * `data` (bytes): data needed to be transmitted;
    * `[timeout]` (int) = 10: timeout of receiving in ms
  
  **Returns**: None if everything ok.

  **Raises** `I2CError` (see error message for more details)
  
* `mem_receive(id, slave_address, memory_address, memory_size, data_length, [timeout] = I2C_DEFAULT_TIME_OUT)`

  Read data given length from slave device's memory by its address
  
  **Args**:

    * `id` (int): see enums;
    * `slave_address` (int, hex): slave device communication address;
    * `memory_address` (int, hex): slave device memory read address;
    * `memory_size` (int, hex): slave device memory read size;
    * `data_length` (int): data length needed to be read;
    * `[timeout]` (int) = 10: timeout of reading in ms
  
  **Returns**: bytes() of given length.

  **Raises** `I2CError` (see error message for more details)
  
* `mem_transmit(id, slave_address, memory_address, memory_size, data, [timeout] = I2C_DEFAULT_TIME_OUT)`

  Write given data to slave device's memory by address
  
  **Args**:

    * `id` (int): see enums;
    * `slave_address` (int, hex): slave device communication address;
    * `memory_address` (int, hex): slave device memory write address;
    * `memory_size` (int, hex): slave device memory write size;
    * `data` (bytes): data needed to be written;
    * `[timeout]` (int) = 10: timeout of writing in ms
  
  **Returns**: None if everything ok.

  **Raises** `I2CError` (see error message for more details)

## Notes ##

* The module halts on fatal errors; create an empty file `.reboot_on_fatal` if a reboot is desired
* The size of micropython heap is roughly 512 Kb. 400k can be realistically allocated right after hard reset.
