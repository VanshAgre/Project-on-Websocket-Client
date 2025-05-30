// Directory: websocket_client_project

// File: .gn
buildconfig = "//build/config/BUILDCONFIG.gn"

// File: BUILD.gn
import("//build/toolchain/toolchain.gni")

toolchain("default") {
}

target("executable", "websocket_client") {
  sources = [
    "src/main.cpp",
    "src/websocket_client.cpp",
    "src/chat_interface.cpp",
    "src/utils.cpp",
  ]

  include_dirs = [
    "src",
    "/usr/include",
    "/usr/local/include"  # Adjust if using vcpkg or custom paths
  ]

  cflags = [ "-std=c++17" ]
  if (is_debug) {
    cflags += [ "-g" ]
  } else {
    cflags += [ "-O2" ]
  }

  libs = [ "pthread", "ssl", "crypto" ]
  lib_dirs = [ "/usr/lib", "/usr/local/lib" ]
}

// File: src/main.cpp
#include "chat_interface.h"
#include <CLI/CLI.hpp>

int main(int argc, char** argv) {
    CLI::App app{"WebSocket Client"};
    std::string uri = "wss://echo.websocket.events";
    app.add_option("-u,--uri", uri, "WebSocket server URI");
    CLI11_PARSE(app, argc, argv);

    ChatInterface chat(uri);
    chat.run();
    return 0;
}

// File: src/websocket_client.h
#pragma once
#include <string>
#include <functional>

class WebSocketClient {
public:
    WebSocketClient(const std::string& uri);
    ~WebSocketClient();

    bool connect();
    void send(const std::string& message);
    void close();

    void onMessage(std::function<void(const std::string&)> handler);

private:
    class Impl;
    Impl* impl_;
};

// File: src/websocket_client.cpp
#include "websocket_client.h"
#include <ixwebsocket/IXWebSocket.h>
#include <iostream>

class WebSocketClient::Impl {
public:
    Impl(const std::string& uri) : uri_(uri), ws_() {
        ws_.setUrl(uri_);
    }

    bool connect() {
        ws_.setOnMessageCallback([this](const ix::WebSocketMessagePtr& msg) {
            if (msg->type == ix::WebSocketMessageType::Message && handler_) {
                handler_(msg->str);
            }
        });
        ws_.start();
        return true;
    }

    void send(const std::string& message) {
        ws_.send(message);
    }

    void close() {
        ws_.stop();
    }

    void onMessage(std::function<void(const std::string&)> handler) {
        handler_ = handler;
    }

private:
    std::string uri_;
    ix::WebSocket ws_;
    std::function<void(const std::string&)> handler_;
};

WebSocketClient::WebSocketClient(const std::string& uri) : impl_(new Impl(uri)) {}
WebSocketClient::~WebSocketClient() { delete impl_; }
bool WebSocketClient::connect() { return impl_->connect(); }
void WebSocketClient::send(const std::string& message) { impl_->send(message); }
void WebSocketClient::close() { impl_->close(); }
void WebSocketClient::onMessage(std::function<void(const std::string&)> handler) {
    impl_->onMessage(handler);
}

// File: src/chat_interface.h
#pragma once
#include <string>

class ChatInterface {
public:
    explicit ChatInterface(const std::string& uri);
    void run();

private:
    std::string uri_;
};

// File: src/chat_interface.cpp
#include "chat_interface.h"
#include "websocket_client.h"
#include <iostream>
#include <string>

ChatInterface::ChatInterface(const std::string& uri) : uri_(uri) {}

void ChatInterface::run() {
    WebSocketClient client(uri_);

    client.onMessage([](const std::string& message) {
        std::cout << "Received: " << message << std::endl;
    });

    if (!client.connect()) {
        std::cerr << "Failed to connect to server." << std::endl;
        return;
    }

    std::string input;
    std::cout << "Connected to " << uri_ << ". Type messages (type 'exit' to quit):\n";
    while (true) {
        std::getline(std::cin, input);
        if (input == "exit") break;
        client.send(input);
    }

    client.close();
}

// File: src/utils.h
#pragma once
#include <string>
#include <ctime>

std::string timestamp();

// File: src/utils.cpp
#include "utils.h"
#include <sstream>
#include <iomanip>

std::string timestamp() {
    auto now = std::time(nullptr);
    std::stringstream ss;
    ss << std::put_time(std::localtime(&now), "%F %T");
    return ss.str();
}

// File: tests/test_websocket.cpp
#define CATCH_CONFIG_MAIN
#include <catch2/catch.hpp>
#include "websocket_client.h"

TEST_CASE("WebSocketClient basic connection", "[websocket]") {
    WebSocketClient client("wss://echo.websocket.events");
    REQUIRE(client.connect());
    client.close();
}
