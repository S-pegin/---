# --- [za_settings.txt] ( https://probot.github.io/ )

￼
เ
>





"For handling zigbee lights
#zigbee_registration_backup.js"
/////////////////////////////////////////////////////////////////////////////////////////////////////////
// Set Initial Variables \\
var fs = reqire('fs');// เชื่อมโยงระบบกลุ่ม
var zmq = ต้องการ ('zeromq'); // กรอบการส่งข้อความแบบอะซิงโครนัส
var matrix_io = ต้องการ ('matrix-protos').matrix_io; // Protocol Buffers สำหรับฟังก์ชัน MATRIX
var matrix_ip = '127.0.0.1';// Local IP
var matrix_zigbee_base_port = 40001;// Port for Zigbee driver
var networkCommands = matrix_io.malos.v1.comm.ZigBeeMsg.NetworkMgmtCmd.NetworkMgmtCmdTypes;// Network Command Types
var networkStatuses = matrix_io.malos.v1.comm.ZigBeeMsg.NetworkMgmtCmd.NetworkStatus// Network Status
var joinTimer = 60// Amount of time for Zigbee devices to join
var gateway_is_active = false; // บูลเพื่อเก็บสถานะเครื่องมือ Gateway CLI

// ขัดให้สร้างกลุ่ม JSON เทียมอุปกรณ์ Zigbee
if (!fs.existsSync('./devices.json')){
  fs.writeFileSync('./devices.json', JSON.stringify({}, null, 2) , 'utf-8');
  console.log('Creating .json file to store Zigbee devices.');
}
// Import Devices.json as an object
console.log('\nLoaded .json file with your Zigbee devices.\n');
var zigbeeDevices = JSON.parse(fs.readFileSync('./devices.json')); // Holds registered Zigbee Devices
// Store device count 
var deviceCount = Object.keys(zigbeeDevices).length;

// สร้างการกำหนดค่าไดรเวอร์สำหรับเครือข่าย Zigbee
var zb_network_msg = matrix_io.malos.v1.driver.DriverConfig.create({
  // Create Zigbee message
  zigbeeMessage: matrix_io.malos.v1.comm.ZigBeeMsg.create({
    // Set type to network management
    type: matrix_io.malos.v1.comm.ZigBeeMsg.ZigBeeCmdType.NETWORK_MGMT,
    // Call a network management command
    networkMgmtCmd: matrix_io.malos.v1.comm.ZigBeeMsg.NetworkMgmtCmd.create({
      // Set command to permit join
      type: matrix_io.malos.v1.comm.ZigBeeMsg.NetworkMgmtCmd.NetworkMgmtCmdTypes.PERMIT_JOIN,
      // Set timer for join period
      permitJoinParams: matrix_io.malos.v1.comm.ZigBeeMsg.NetworkMgmtCmd.PermitJoinParams.create({time: joinTimer})
    })
  })
});

/////////////////////////////////////////////////////////////////////////////////////////////////////////
// BASE PORT \\
// Create a Pusher socket
var configSocket = zmq.socket('push');
// Connect Pusher to Base port
configSocket.connect('tcp://' + matrix_ip + ':' + matrix_zigbee_base_port);
// Create driver configuration for updates/timeouts
var config = matrix_io.malos.v1.driver.DriverConfig.create({
  // Update rate configuration
  delayBetweenUpdates: 1.0,// 2 seconds between updates
  timeoutAfterLastPing: 6.0,// Stop sending updates 6 seconds after pings.
});
// Send initial driver configuration
configSocket.send(matrix_io.malos.v1.driver.DriverConfig.encode(config).finish());

// Reset Gateway CLI Tool
resetGateway(
  // Wait 3 seconds
  setTimeout(function(){
    // Request Gateway status in Data port
    isGatewayActive();
  }, 2000)
);

/////////////////////////////////////////////////////////////////////////////////////////////////////////
// KEEP-ALIVE PORT \\
// Create a Pusher socket
var pingSocket = zmq.socket('push');
// Connect Pusher to Keep-alive port
pingSocket.connect('tcp://' + matrix_ip + ':' + (matrix_zigbee_base_port + 1));
// Send initial ping
pingSocket.send('');
// Send a ping every second
setInterval(function(){
  pingSocket.send('');
}, 1000);

/////////////////////////////////////////////////////////////////////////////////////////////////////////
// ERROR PORT \\
// Create a Subscriber socket
var errorSocket = zmq.socket('sub');
// Connect Subscriber to Error port
errorSocket.connect('tcp://' + matrix_ip + ':' + (matrix_zigbee_base_port + 2));
// Connect Subscriber to Error port
errorSocket.subscribe('');
// On Message
errorSocket.on('message', function(error_message){
  console.log('Received Message: ' + error_message.toString('utf8'));// Log error
});

/////////////////////////////////////////////////////////////////////////////////////////////////////////
// DATA UPDATE PORT \\
// Create a Subscriber socket
var updateSocket = zmq.socket('sub');
// Connect Subscriber to Data Update port
updateSocket.connect('tcp://' + matrix_ip + ':' + (matrix_zigbee_base_port + 3));
// Subscribe to messages
updateSocket.subscribe('');
// On Message
updateSocket.on('message', function(buffer){
  var data = matrix_io.malos.v1.comm.ZigBeeMsg.decode(buffer);// Extract message
  // If gateway active and devices are waiting to join
  if(gateway_is_active && data.networkMgmtCmd.type === networkCommands.DISCOVERY_INFO){
    // Manage Devices connecting
    manageZigbeeDevices(buffer);
  }
  // If gateway active and network status is requested
  else if(gateway_is_active && zb_network_msg.zigbeeMessage.networkMgmtCmd.type === networkCommands.NETWORK_STATUS){
    // Switch Cases For Network Statuses
    switch(data.networkMgmtCmd.networkStatus.type){
      //* IF NO NETWORK
      case networkStatuses.Status.NO_NETWORK:
        console.log('No Network');
        // Add (create network) to configuration
        zb_network_msg.zigbeeMessage.networkMgmtCmd.type = networkCommands.CREATE_NWK;
        // Send configuration
        configSocket.send(matrix_io.malos.v1.driver.DriverConfig.encode(zb_network_msg).finish());
        break;
      //* IF JOINING NETWORK
      case networkStatuses.Status.JOINING_NETWORK:
        console.log('Joining Network');
        break;
      //* IF JOINED NETWORK
      case networkStatuses.Status.JOINED_NETWORK:
        console.log('Joined Existing Network');
        // Add (permit devices to join) in configuration
        zb_network_msg.zigbeeMessage.networkMgmtCmd.type = networkCommands.PERMIT_JOIN;
        // Add (set join time limit to 60 seconds) in configuration
        zb_network_msg.zigbeeMessage.networkMgmtCmd.permitJoinParams.time = joinTimer;
        // Send configuration
        configSocket.send(matrix_io.malos.v1.driver.DriverConfig.encode(zb_network_msg).finish());
        // Log status
        console.log('Waiting ' + joinTimer + ' seconds for devices to join.');
        // Start timer to exit & save program
        saveAndQuit(joinTimer);
        break;
      //* IF JOINED NETWORK WITH NO PARENT
      case networkStatuses.Status.JOINED_NETWORK_NO_PARENT:
        console.log('Joined Network With No Parent');
        break;
      //* IF LEAVING NETWORK
      case networkStatuses.Status.LEAVING_NETWORK:
        console.log('Leaving Network');
        break;
    }
  }
  // Check if Gateway tool restarted
  else if(gateway_is_active === false){
    gateway_is_active = true;// update boolean
    // If Gateway tool is active
    if(data.networkMgmtCmd.isProxyActive){
      console.log('Gateway CLI Tool is active.');// Log status
      // Add request for Zigbee Network Status in configuration
      zb_network_msg.zigbeeMessage.networkMgmtCmd.type = networkCommands.NETWORK_STATUS;
      // Send configuration
      configSocket.send(matrix_io.malos.v1.driver.DriverConfig.encode(zb_network_msg).finish());
    }
    // If Gateway Tool is down
    else{
      console.log('\nGateway Reset Failed. Try restarting.');// Log status
      process.exit(1);// Exit application
    }
  }
});

/////////////////////////////////////////////////////////////////////////////////////////////////////////
// FUNCTIONS \\
// - Restart Zigbee CLI tool called Gateway (optional, but ensures tool is running)
function resetGateway(callback) {
  console.log('Resetting Gateway CLI Tool.');
  // Define configuration message as Reset
  zb_network_msg.zigbeeMessage.networkMgmtCmd.type = matrix_io.malos.v1.comm.ZigBeeMsg.NetworkMgmtCmd.NetworkMgmtCmdTypes.RESET_PROXY;
  // Send configuration to Base Port
  configSocket.send(matrix_io.malos.v1.driver.DriverConfig.encode(zb_network_msg).finish());
  // Run callback if defined
  if(callback)
    callback;
}
// - Ask for Gateway status through Data port
function isGatewayActive() {
  // Log that connection is being tested
  console.log('Checking connection with the Gateway CLI Tool.');
  // Save Gateway status request to configuration 
  zb_network_msg.zigbeeMessage.networkMgmtCmd.type = matrix_io.malos.v1.comm.ZigBeeMsg.NetworkMgmtCmd.NetworkMgmtCmdTypes.IS_PROXY_ACTIVE;
  // Send configuration to Base port
  configSocket.send(matrix_io.malos.v1.driver.DriverConfig.encode(zb_network_msg).finish());
}

// - Save & Quit program
function saveAndQuit(timer){
  // Counter to count each second
  var counter = 0;
  // Increase counter every second
  setInterval(function(){
    // If counter has reached timer, exit
    if(counter >= timer){
      console.log('\n'+joinTimer+' seconds passed. Saving&Exiting program.');
      process.exit();// exit application
    }
    // Else increase counter
    else{
      counter++;
      console.log(counter);
    }
  },1000);
}

// - Save discovered devices
function manageZigbeeDevices(buffer){
  console.log('Device(s) found!');
  // Extract Data Update Port message
  var data = matrix_io.malos.v1.comm.ZigBeeMsg.decode(buffer);
  //Look for Zigbee devices with an	ON/OFF cluster ID
  // For each node
  data.networkMgmtCmd.connectedNodes.map(function(nodes){
    console.log(nodes);
    // For each endpoint in nodes
    nodes.endpoints.map(function(endpoint){
      // For each cluster in endpoint
      endpoint.clusters.map(function(cluster){

        // For each saved device
        for(device in zigbeeDevices){
          // If newly discovered device was already saved
          if( zigbeeDevices[device].node_id === nodes.nodeId )
            return;// Exit function
        }

        // If cluster ID is ON/OFF
        if (cluster.clusterId == 6) {
          // Save device nodeId & endpointIndex to JSON file
          zigbeeDevices['device_'+deviceCount] = {node_id:nodes.nodeId, endpoint_index: endpoint.endpointIndex};
          // Update device count
          deviceCount++;
          // Update devices.json
          fs.writeFile('./devices.json', JSON.stringify(zigbeeDevices, null, 2) , 'utf-8', function(){
            console.log('Saved discovered device');// Log that device was saved
          });
        }

      });
    });
  });
}
zigbee_toggle_backup.js
/////////////////////////////////////////////////////////////////////////////////////////////////////////
// Set Initial Variables \\
var fs = require('fs');// File system library
var zmq = require('zeromq');// Asynchronous Messaging Framework
var matrix_io = require('matrix-protos').matrix_io;// Protocol Buffers for MATRIX function
var matrix_ip = '127.0.0.1';// Local IP
var matrix_zigbee_base_port = 40001;// Port for Zigbee driver
var networkCommands = matrix_io.malos.v1.comm.ZigBeeMsg.NetworkMgmtCmd.NetworkMgmtCmdTypes;// Network Command Types
var networkStatuses = matrix_io.malos.v1.comm.ZigBeeMsg.NetworkMgmtCmd.NetworkStatus// Network Status
var gateway_is_active = false;// Bool to hold Gateway CLI Tool ON status
var gateway_is_restarting = false// Bool to hold Gateway CLI Tool ON status
// If missing, create JSON file to store Zigbee devices
if (!fs.existsSync('./devices.json')){
  console.log('devices.json was not found!');
  process.exit(1);
}
// Import Devices.json as an object
console.log('\nLoaded .json file with your Zigbee devices.\n');
var zigbeeDevices = JSON.parse(fs.readFileSync('./devices.json')); // Holds registered Zigbee Devices

// Create driver configuration for Zigbee network
var zb_network_msg = matrix_io.malos.v1.driver.DriverConfig.create({
    zigbeeMessage: matrix_io.malos.v1.comm.ZigBeeMsg.create({
      type: matrix_io.malos.v1.comm.ZigBeeMsg.ZigBeeCmdType.NETWORK_MGMT,
      networkMgmtCmd: matrix_io.malos.v1.comm.ZigBeeMsg.NetworkMgmtCmd.create({
        type: matrix_io.malos.v1.comm.ZigBeeMsg.NetworkMgmtCmd.NetworkMgmtCmdTypes.PERMIT_JOIN,
      })
    })
  });

/////////////////////////////////////////////////////////////////////////////////////////////////////////
// BASE PORT \\
// Create a Pusher socket
var configSocket = zmq.socket('push');
// Connect Pusher to Base port
configSocket.connect('tcp://' + matrix_ip + ':' + matrix_zigbee_base_port);
// Create driver configuration for updates/timeouts
var config = matrix_io.malos.v1.driver.DriverConfig.create({
  // Update rate configuration
  delayBetweenUpdates: 1.0,// 2 seconds between updates
  timeoutAfterLastPing: 6.0,// Stop sending updates 6 seconds after last ping.
});
// Send initial driver configuration
configSocket.send(matrix_io.malos.v1.driver.DriverConfig.encode(config).finish());

setTimeout(isGatewayActive, 3000);


/////////////////////////////////////////////////////////////////////////////////////////////////////////
// KEEP-ALIVE PORT \\
// Create a Pusher socket
var pingSocket = zmq.socket('push');
// Connect Pusher to Keep-alive port
pingSocket.connect('tcp://' + matrix_ip + ':' + (matrix_zigbee_base_port + 1));
// Send initial ping
pingSocket.send('');
// Send a ping every second
setInterval(function(){
  pingSocket.send('');
}, 1000);

/////////////////////////////////////////////////////////////////////////////////////////////////////////
// ERROR PORT \\
// Create a Subscriber socket
var errorSocket = zmq.socket('sub');
// Connect Subscriber to Error port
errorSocket.connect('tcp://' + matrix_ip + ':' + (matrix_zigbee_base_port + 2));
// Connect Subscriber to Error port
errorSocket.subscribe('');
// On Message
errorSocket.on('message', function(error_message){
  console.log('Received Message: ' + error_message.toString('utf8'));// Log error
});

/////////////////////////////////////////////////////////////////////////////////////////////////////////
// DATA UPDATE PORT \\
// Create a Subscriber socket
var updateSocket = zmq.socket('sub');
// Connect Subscriber to Data Update port
updateSocket.connect('tcp://' + matrix_ip + ':' + (matrix_zigbee_base_port + 3));
// Subscribe to messages
updateSocket.subscribe('');
// On Message
updateSocket.on('message', function(buffer){
  var data = matrix_io.malos.v1.comm.ZigBeeMsg.decode(buffer);// Extract message
  //console.log(data);
  // If gateway active and network status is requested
  if(gateway_is_active && zb_network_msg.zigbeeMessage.networkMgmtCmd.type === networkCommands.NETWORK_STATUS){
    // Switch Cases For Network Statuses
    switch(data.networkMgmtCmd.networkStatus.type){
      //* IF NO NETWORK
      case networkStatuses.Status.NO_NETWORK:
        console.log('No Network');
        process.exit(1);// Exit application
        break;
      //* IF JOINING NETWORK
      case networkStatuses.Status.JOINING_NETWORK:
        console.log('Joining Network');
        break;
      //* IF JOINED NETWORK
      case networkStatuses.Status.JOINED_NETWORK:
        console.log('Joined Existing Network\n');
        sendAllToggleCommand();
        break;
    }
  }
  // Check if Gateway tool restarted
  else if(gateway_is_active === false){
    // If Gateway tool is active
    if(data.networkMgmtCmd.isProxyActive){
      gateway_is_active = true;// update boolean
      console.log('Gateway CLI Tool is active.');// Log status
      // Add request for Zigbee Network Status in configuration
      zb_network_msg.zigbeeMessage.networkMgmtCmd.type = networkCommands.NETWORK_STATUS;
      // Send configuration
      configSocket.send(matrix_io.malos.v1.driver.DriverConfig.encode(zb_network_msg).finish());
    }
    // If Gateway CLI Tool is down
    else if (gateway_is_restarting === false){
      gateway_is_restarting = true;// update boolean
      console.log('Gateway CLI Tool Is Offline. Please wait 10 seconds for restart.');// Log status
      resetGateway( setTimeout(isGatewayActive, 10000) );// Restart Gateway
    }
  }
});

/////////////////////////////////////////////////////////////////////////////////////////////////////////
// FUNCTIONS \\
// - Restart Zigbee CLI tool called Gateway (optional, but ensures tool is running)
function resetGateway(callback) {
  console.log('Restarting Gateway Tool.\n');
  // Define configuration message as Reset
  zb_network_msg.zigbeeMessage.networkMgmtCmd.type = matrix_io.malos.v1.comm.ZigBeeMsg.NetworkMgmtCmd.NetworkMgmtCmdTypes.RESET_PROXY;
  // Send configuration to Base Port
  configSocket.send(matrix_io.malos.v1.driver.DriverConfig.encode(zb_network_msg).finish());
  // Run callback if defined
  if(callback)
    callback;
}
// - Ask for Gateway status through Data port
function isGatewayActive() {
  // Log that connection is being tested
  console.log('Checking connection with the Gateway');
  // Save Gateway status request to configuration 
  zb_network_msg.zigbeeMessage.networkMgmtCmd.type = matrix_io.malos.v1.comm.ZigBeeMsg.NetworkMgmtCmd.NetworkMgmtCmdTypes.IS_PROXY_ACTIVE;
  // Send configuration to Base port
  configSocket.send(matrix_io.malos.v1.driver.DriverConfig.encode(zb_network_msg).finish());
}

// - Toggle each zigbee device on & off
function sendAllToggleCommand(){
  setInterval(function(){


  for(device in zigbeeDevices){
    console.log('Sent message to ' + device);

var zb_toggle_msg = matrix_io.malos.v1.driver.DriverConfig.create({
  zigbeeMessage: matrix_io.malos.v1.comm.ZigBeeMsg.create({
    type: matrix_io.malos.v1.comm.ZigBeeMsg.ZigBeeCmdType.ZCL,
    zclCmd: matrix_io.malos.v1.comm.ZigBeeMsg.ZCLCmd.create({
      type: matrix_io.malos.v1.comm.ZigBeeMsg.ZCLCmd.OnOffCmd.ZCLOnOffCmdType.ON_OFF,
      onoffCmd: matrix_io.malos.v1.comm.ZigBeeMsg.ZCLCmd.OnOffCmd.create({
        type: matrix_io.malos.v1.comm.ZigBeeMsg.ZCLCmd.OnOffCmd.ZCLOnOffCmdType.TOGGLE
      }),
      nodeId: zigbeeDevices[device].node_id,
      endpointIndex: zigbeeDevices[device].endpoint_index
    })
  })
});
    configSocket.send(matrix_io.malos.v1.driver.DriverConfig.encode(zb_toggle_msg).finish());
  }



  },2000);
}
@S-pegin
 
Add heading textAdd bold text, <Ctrl+b>Add italic text, <Ctrl+i>
Add a quote, <Ctrl+Shift+.>Add code, <Ctrl+e>Add a link, <Ctrl+k>
Add a bulleted list, <Ctrl+Shift+8>Add a numbered list, <Ctrl+Shift+7>Add a task list, <Ctrl+Shift+l>
Directly mention a user or team
Reference an issue or pull request
Leave a comment
ไม่ได้เลือกไฟล์ใด
Attach files by dragging & dropping, selecting or pasting them.
Styling with Markdown is supported
Footer
© 2022 GitHub, Inc.
Footer navigation
Terms
Privacy
Security
Status
Docs
Contact GitHub
Pricing
API
Training
Blog
About
หน่วยงานกำกับดูแลไฟ zigbeezigbee.
สถานะวิ่งคิว

{ "avgStepBuildTime" : 64.9566, "avgStepTime" : 94.4354, "buildReadTimeAvgMs" : 456.574, "buildReadTimeMs" : 2047729724, "bytesReceived" : 40958754227093, "bytesSent" : 1189537]
