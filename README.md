# Shelly Relay Controller with Klipper Integration

Relay on/off control:

When you press the button and the relay is off, the relay will be activated.
When you press the button and the relay is already turned on, a command will be sent to Klipper for safe shutdown and the relay will turn off after 28 seconds automatically.

Integration with Klipper:

When the relay is on and the button is pressed, the command 
http://YourKlipperIP:7125/printer/gcode/script?script=SHUTDOWN will be sent to Klipper. This command can be used to execute a macro or a specific action on the 3D printer managed by Klipper.

Add the following macro on your klipper:


```
[gcode_macro SHUTDOWN]
gcode:
    UPDATE_DELAYED_GCODE ID=SHUTDOWNRPI DURATION=8


[delayed_gcode SHUTDOWNRPI]
gcode:
    {action_call_remote_method("shutdown_machine")}"
```

# Configure  Output Settings on Shelly:
![image](https://github.com/user-attachments/assets/671ef0fc-777f-4038-bd12-3508478ad041)



# Create a new script on Shelly: 

![image](https://github.com/user-attachments/assets/89254e8e-ff73-4caa-99e2-88be873d6eb4)

# Click add script and paste the following code (Replace "YourKlipperIP" with your printer's IP address):

```
let CONFIG = {
  toggleTimeout: 28 * 1000, // Timer di 28 secondi in millisecondi
  inputId: 0,
  switchId: 0,
  klipperUrl: "http://YourKlipperIP:7125/printer/gcode/script?script=SHUTDOWN",
  lastEventTime: 0, // Tempo dell'ultimo evento per evitare chiamate multiple
  debounceTime: 1000, // Tempo di debounce in millisecondi per evitare ripetizioni
  relayTimerHandle: null // Handle per il timer del relay
};

// Configurazione del relay in modalità "detached"
Shelly.call("Switch.SetConfig", {
  id: CONFIG.switchId,
  config: {
    in_mode: "detached",
  },
});

// Funzione per inviare il comando a Klipper
function sendKlipperCommand() {
  print("Invio comando a Klipper...");
  Shelly.call(
    "http.request",
    {
      method: "GET",
      url: CONFIG.klipperUrl
    },
    function (response, error_code, error_message) {
      if (error_code === 0) {
        print("Comando Klipper inviato con successo.");
      } else {
        print("Invio comando a Klipper fallito: " + error_message);
      }
    }
  );
}

// Funzione per spegnere il relay dopo un ritardo
function scheduleRelayOff() {
  // Cancella il timer esistente se presente
  if (CONFIG.relayTimerHandle !== null) {
    Timer.clear(CONFIG.relayTimerHandle);
  }

  // Imposta un nuovo timer per spegnere il relay
  CONFIG.relayTimerHandle = Timer.set(CONFIG.toggleTimeout, false, function() {
    print("Spengo il relay dopo 28 secondi.");
    Shelly.call("Switch.Set", { id: CONFIG.switchId, on: false });
  });
}

// Gestore degli eventi per il pulsante
Shelly.addEventHandler(function (event) {
  let now = Date.now();

  if (typeof event.info.event === "undefined") return;

  if (event.info.component === "input:" + CONFIG.inputId) {
    // Evita azioni ripetute entro il tempo di debounce
    if (now - CONFIG.lastEventTime < CONFIG.debounceTime) return;

    CONFIG.lastEventTime = now;
    print("Evento pulsante rilevato: " + event.info.event);

    Shelly.call("Switch.GetStatus", { id: CONFIG.switchId }, function (status, err_code, err_msg) {
      if (err_code === 0) {
        if (status.output === true) {
          // Il relay è acceso
          print("Relay è ACCESO. Invio comando a Klipper e imposto timer per spegnere il relay.");
          sendKlipperCommand();
          scheduleRelayOff();
        } else {
          // Il relay è spento, accendilo
          print("Relay è SPENTO. Accensione relay.");
          Shelly.call("Switch.Set", { id: CONFIG.switchId, on: true });
        }
      } else {
        print("Errore nel recuperare lo stato del relay: " + err_msg);
      }
    });
  }
});
```
# Give the script a name and save it and then run it

![image](https://github.com/user-attachments/assets/fb383cc5-3a29-46c1-80a0-5fd3ad428898)


# Make sure the script is activated and running
![image](https://github.com/user-attachments/assets/365a3ab8-63d0-48c7-af7d-6fde47406889)


# How to Use
Prepare the Device:

![image](https://github.com/user-attachments/assets/f774a28f-0ee5-4bd6-bf5f-869bb75b3a4e)


Ensure that the Shelly device is set up and connected to the relay and button as described.

# Configure the Script:

Modify the following variables in the code to match your environment:

inputId: ID of the connected button.

switchId: ID of the relay to control.

klipperUrl: URL of the command to be sent to Klipper.



# Upload the Script:

Upload the script to your Shelly device via the web interface or the Shelly app.



# Test the Script:

Verify the functionality by pressing the button and ensuring that the relay and Klipper command operate as expected.






