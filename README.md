<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>HTML Hack Client UI</title>
    <style>
        body { background: #1a1a1a; color: white; font-family: sans-serif; text-align: center; padding: 50px; }
        .btn { background: #ff4444; border: none; color: white; padding: 15px 32px; font-size: 16px; margin: 10px; cursor: pointer; border-radius: 5px; transition: 0.3s; }
        .btn.active { background: #00c851; }
    </style>
</head>
<body>
    <h1>HTML Client Dashboard</h1>
    <button id="flightBtn" class="btn" onclick="toggleHack('flight')">Toggle Flight</button>
    <button id="auraBtn" class="btn" onclick="toggleHack('killaura')">Toggle KillAura</button>

    <script>
        function toggleHack(hackName) {
            fetch(`http://localhost:8080/toggle?module=${hackName}`, { method: 'POST' })
            .then(response => response.text())
            .then(data => {
                const btn = document.getElementById(hackName === 'flight' ? 'flightBtn' : 'auraBtn');
                btn.classList.toggle('active');
            })
            .catch(err => console.error('Error communicating with Minecraft:', err));
        }
    </script>
</body>
</html>
package com.example.client.server;

import com.example.client.CustomClient;
import com.example.client.modules.Module;
import com.sun.net.httpserver.HttpExchange;
import com.sun.net.httpserver.HttpHandler;
import com.sun.net.httpserver.HttpServer;
import java.io.IOException;
import java.io.OutputStream;
import java.net.InetSocketAddress;

public class ClientHttpServer {
    public static void startServer() {
        try {
            // Spin up a local server on port 8080
            HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);
            server.createContext("/toggle", new ToggleHandler());
            server.setExecutor(null); 
            server.start();
            System.out.println("HTML Client Server started on port 8080");
        } catch (IOException e) {
            e.printStackTrace;
        }
    }

    static class ToggleHandler implements HttpHandler {
        @Override
        public void handle(HttpExchange exchange) throws IOException {
            // Handle CORS preflight security checks from the browser
            exchange.getResponseHeaders().add("Access-Control-Allow-Origin", "*");
            exchange.getResponseHeaders().add("Access-Control-Allow-Methods", "POST, GET, OPTIONS");
            
            if ("OPTIONS".equalsIgnoreCase(exchange.getRequestMethod())) {
                exchange.sendResponseHeaders(204, -1);
                return;
            }

            // Extract query parameter ?module=
            String query = exchange.getRequestURI().getQuery();
            String response = "Invalid Request";
            
            if (query != null && query.contains("module=")) {
                String moduleName = query.split("=")[1];
                
                // Interact with the core Java game client logic
                Module module = CustomClient.instance.moduleManager.getModuleByName(moduleName);
                if (module != null) {
                    module.toggle();
                    response = "Module " + moduleName + " status: " + module.isToggled();
                }
            }

            // Return status message back to the HTML page
            exchange.sendResponseHeaders(200, response.length());
            OutputStream os = exchange.getResponseBody();
            os.write(response.getBytes());
            os.close();
        }
    }
}
package com.example.client;

import com.example.client.modules.ModuleManager;
import com.example.client.server.ClientHttpServer;
import net.minecraftforge.fml.common.Mod;
import net.minecraftforge.fml.common.event.FMLInitializationEvent;

@Mod(modid = "htmlclient", name = "HTML Client Backend", version = "1.0")
public class CustomClient {
    public static CustomClient instance;
    public ModuleManager moduleManager;

    @Mod.EventHandler
    public void init(FMLInitializationEvent event) {
        instance = this;
        this.moduleManager = new ModuleManager();
        
        // Boot up the API listener on a dedicated thread
        new Thread(ClientHttpServer::startServer).start();
    }
}
