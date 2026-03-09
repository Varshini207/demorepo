# Remote Task Manager (WinUI Frontend + Win32 Backend) in VS Code

This guide builds a **working MVP** remote task manager on Windows:
- **Backend (Win32/C++)**: native HTTP server (`/tasks` CRUD) using Winsock + simple JSON handling.
- **Frontend (WinUI 3/C#)**: desktop app that calls backend API and displays/edits tasks.

---

## 1) Prerequisites

Install on Windows:
1. **Visual Studio Build Tools 2022** with:
   - MSVC v143
   - Windows 10/11 SDK
   - C++ CMake tools
2. **.NET 8 SDK**
3. **VS Code** + extensions:
   - C/C++ (Microsoft)
   - CMake Tools
   - C# Dev Kit
   - XAML (optional but useful)
4. **Git**
5. **Windows App SDK runtime** (for WinUI 3)

Open **Developer PowerShell for VS 2022** before building C++ backend.

### 1.1 Exact pre-run checklist (do this before any code execution)

Use this checklist exactly in order:

1. **Use a Windows machine** (Windows 10/11, 64-bit).
   - This project will not run correctly from Linux/macOS because WinUI 3 and Win32 toolchains are Windows-specific.

2. **Install Visual Studio Build Tools 2022**.
   - During installation, select C++ build components:
     - MSVC v143 build tools
     - Windows 10 or 11 SDK
     - C++ CMake tools for Windows

3. **Install .NET 8 SDK**.
   - Needed to build and run the WinUI frontend.

4. **Install VS Code**.
   - Add extensions:
     - C/C++
     - CMake Tools
     - C# Dev Kit
     - XAML (recommended)

5. **Install Git**.
   - Needed to clone/open project and manage updates.

6. **Install Windows App SDK Runtime**.
   - Required by WinUI 3 apps at runtime.

7. **Open the correct terminal**:
   - Open **Developer PowerShell for VS 2022**.
   - Run `code .` from that terminal so VS Code inherits MSVC environment variables.

8. **Verify tool installation** (run inside terminal in VS Code):

```powershell
cl
cmake --version
dotnet --version
git --version
```

If any command is missing, install/fix that dependency before continuing.

### 1.2 Where to run each part

- **Backend build/run**: run from the `backend-win32` folder in terminal.
- **Frontend build/run**: run from the `frontend-winui` folder in terminal.
- **API checks (`curl`)**: run from any folder, as long as backend is running.

### 1.3 Minimum ports/network requirements

- Backend listens on **`http://localhost:8080`**.
- Ensure no other process uses port `8080`.
- If Windows Firewall prompts, allow local access.

---

## 2) Workspace layout

```text
remote-task-manager/
  backend-win32/
    CMakeLists.txt
    src/
      main.cpp
      task_store.h
      task_store.cpp
      http_server.h
      http_server.cpp
  frontend-winui/
    RemoteTaskManager.UI.csproj
    App.xaml
    App.xaml.cs
    MainWindow.xaml
    MainWindow.xaml.cs
    Models/TaskItem.cs
    Services/TaskApiClient.cs
```

---

## 3) Backend (Win32 C++) code

### 3.1 `backend-win32/CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.20)
project(RemoteTaskManagerBackend LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(RemoteTaskManagerBackend
    src/main.cpp
    src/task_store.cpp
    src/http_server.cpp
)

target_include_directories(RemoteTaskManagerBackend PRIVATE src)
target_link_libraries(RemoteTaskManagerBackend PRIVATE ws2_32)
```

### 3.2 `backend-win32/src/task_store.h`

```cpp
#pragma once
#include <mutex>
#include <optional>
#include <string>
#include <vector>

struct Task {
    int id{};
    std::string title;
    bool completed{};
};

class TaskStore {
public:
    Task create(const std::string& title);
    std::vector<Task> all() const;
    std::optional<Task> update(int id, const std::string& title, bool completed);
    bool remove(int id);

private:
    mutable std::mutex mtx_;
    std::vector<Task> tasks_;
    int nextId_ = 1;
};
```

### 3.3 `backend-win32/src/task_store.cpp`

```cpp
#include "task_store.h"

Task TaskStore::create(const std::string& title) {
    std::lock_guard lock(mtx_);
    Task t{nextId_++, title, false};
    tasks_.push_back(t);
    return t;
}

std::vector<Task> TaskStore::all() const {
    std::lock_guard lock(mtx_);
    return tasks_;
}

std::optional<Task> TaskStore::update(int id, const std::string& title, bool completed) {
    std::lock_guard lock(mtx_);
    for (auto& t : tasks_) {
        if (t.id == id) {
            t.title = title;
            t.completed = completed;
            return t;
        }
    }
    return std::nullopt;
}

bool TaskStore::remove(int id) {
    std::lock_guard lock(mtx_);
    auto before = tasks_.size();
    tasks_.erase(
        std::remove_if(tasks_.begin(), tasks_.end(), [id](const Task& t) { return t.id == id; }),
        tasks_.end());
    return tasks_.size() != before;
}
```

### 3.4 `backend-win32/src/http_server.h`

```cpp
#pragma once
#include "task_store.h"

class HttpServer {
public:
    explicit HttpServer(TaskStore& store);
    void run(unsigned short port);

private:
    TaskStore& store_;
};
```

### 3.5 `backend-win32/src/http_server.cpp`

```cpp
#include "http_server.h"
#include <algorithm>
#include <array>
#include <cstring>
#include <iostream>
#include <sstream>
#include <string>
#include <winsock2.h>
#include <ws2tcpip.h>

static std::string jsonEscape(const std::string& s) {
    std::string out;
    for (char c : s) {
        if (c == '"') out += "\\\"";
        else if (c == '\\') out += "\\\\";
        else out += c;
    }
    return out;
}

static std::string taskToJson(const Task& t) {
    std::ostringstream o;
    o << "{\"id\":" << t.id
      << ",\"title\":\"" << jsonEscape(t.title)
      << "\",\"completed\":" << (t.completed ? "true" : "false") << "}";
    return o.str();
}

static std::string tasksToJson(const std::vector<Task>& tasks) {
    std::ostringstream o;
    o << "[";
    for (size_t i = 0; i < tasks.size(); ++i) {
        if (i) o << ",";
        o << taskToJson(tasks[i]);
    }
    o << "]";
    return o.str();
}

static std::string response(int code, const std::string& status, const std::string& body,
                            const std::string& contentType = "application/json") {
    std::ostringstream o;
    o << "HTTP/1.1 " << code << " " << status << "\r\n"
      << "Content-Type: " << contentType << "\r\n"
      << "Access-Control-Allow-Origin: *\r\n"
      << "Access-Control-Allow-Methods: GET,POST,PUT,DELETE,OPTIONS\r\n"
      << "Access-Control-Allow-Headers: Content-Type\r\n"
      << "Content-Length: " << body.size() << "\r\n\r\n"
      << body;
    return o.str();
}

// NOTE: Minimal parser for demo only.
static std::string extract(const std::string& body, const std::string& key) {
    auto p = body.find("\"" + key + "\"");
    if (p == std::string::npos) return {};
    p = body.find(':', p);
    if (p == std::string::npos) return {};
    auto start = body.find_first_not_of(" \t\n\r\"", p + 1);
    if (start == std::string::npos) return {};

    if (body[start] == 't' || body[start] == 'f') {
        auto end = body.find_first_of(",}\r\n", start);
        return body.substr(start, end - start);
    }

    auto end = body.find('"', start);
    if (end != std::string::npos) return body.substr(start, end - start);

    end = body.find_first_of(",}\r\n", start);
    return body.substr(start, end - start);
}

HttpServer::HttpServer(TaskStore& store) : store_(store) {}

void HttpServer::run(unsigned short port) {
    WSADATA wsa{};
    WSAStartup(MAKEWORD(2, 2), &wsa);

    SOCKET server = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    addr.sin_addr.s_addr = INADDR_ANY;

    bind(server, reinterpret_cast<sockaddr*>(&addr), sizeof(addr));
    listen(server, SOMAXCONN);

    std::cout << "Backend listening on http://0.0.0.0:" << port << "\n";

    while (true) {
        SOCKET client = accept(server, nullptr, nullptr);
        if (client == INVALID_SOCKET) continue;

        std::array<char, 8192> buf{};
        int n = recv(client, buf.data(), static_cast<int>(buf.size()), 0);
        if (n <= 0) {
            closesocket(client);
            continue;
        }

        std::string req(buf.data(), n);
        std::istringstream in(req);
        std::string method, path, version;
        in >> method >> path >> version;

        auto bodyPos = req.find("\r\n\r\n");
        std::string body = bodyPos == std::string::npos ? "" : req.substr(bodyPos + 4);

        std::string out;

        if (method == "OPTIONS") {
            out = response(200, "OK", "{}");
        } else if (method == "GET" && path == "/tasks") {
            out = response(200, "OK", tasksToJson(store_.all()));
        } else if (method == "POST" && path == "/tasks") {
            auto title = extract(body, "title");
            if (title.empty()) {
                out = response(400, "Bad Request", "{\"error\":\"title required\"}");
            } else {
                auto t = store_.create(title);
                out = response(201, "Created", taskToJson(t));
            }
        } else if (method == "PUT" && path.rfind("/tasks/", 0) == 0) {
            int id = std::stoi(path.substr(7));
            auto title = extract(body, "title");
            auto completedStr = extract(body, "completed");
            bool completed = (completedStr == "true");
            auto updated = store_.update(id, title, completed);
            out = updated
                ? response(200, "OK", taskToJson(*updated))
                : response(404, "Not Found", "{\"error\":\"not found\"}");
        } else if (method == "DELETE" && path.rfind("/tasks/", 0) == 0) {
            int id = std::stoi(path.substr(7));
            out = store_.remove(id)
                ? response(200, "OK", "{\"ok\":true}")
                : response(404, "Not Found", "{\"error\":\"not found\"}");
        } else {
            out = response(404, "Not Found", "{\"error\":\"route\"}");
        }

        send(client, out.c_str(), static_cast<int>(out.size()), 0);
        closesocket(client);
    }
}
```

### 3.6 `backend-win32/src/main.cpp`

```cpp
#include "http_server.h"
#include "task_store.h"

int main() {
    TaskStore store;
    HttpServer server(store);
    server.run(8080);
    return 0;
}
```

### 3.7 Build and run backend

```powershell
cd backend-win32
cmake -S . -B build -G "Ninja"
cmake --build build
.\build\RemoteTaskManagerBackend.exe
```

Test quickly:

```powershell
curl http://localhost:8080/tasks
curl -X POST http://localhost:8080/tasks -H "Content-Type: application/json" -d '{"title":"First task"}'
curl http://localhost:8080/tasks
```

---

## 4) Frontend (WinUI 3 + C#) code

> Create this as a WinUI 3 desktop app (packaged) project in VS Code using `dotnet new` templates (after installing WinUI templates), or create once in Visual Studio and then open in VS Code.

### 4.1 `frontend-winui/RemoteTaskManager.UI.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows10.0.19041.0</TargetFramework>
    <TargetPlatformMinVersion>10.0.19041.0</TargetPlatformMinVersion>
    <RootNamespace>RemoteTaskManager.UI</RootNamespace>
    <UseWinUI>true</UseWinUI>
    <EnableMsixTooling>true</EnableMsixTooling>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.WindowsAppSDK" Version="1.5.240311000" />
  </ItemGroup>
</Project>
```

### 4.2 `frontend-winui/App.xaml`

```xml
<Application
    x:Class="RemoteTaskManager.UI.App"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Application.Resources />
</Application>
```

### 4.3 `frontend-winui/App.xaml.cs`

```csharp
using Microsoft.UI.Xaml;

namespace RemoteTaskManager.UI;

public partial class App : Application
{
    private Window? _window;

    public App()
    {
        this.InitializeComponent();
    }

    protected override void OnLaunched(LaunchActivatedEventArgs args)
    {
        _window = new MainWindow();
        _window.Activate();
    }
}
```

### 4.4 `frontend-winui/Models/TaskItem.cs`

```csharp
namespace RemoteTaskManager.UI.Models;

public class TaskItem
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public bool Completed { get; set; }
}
```

### 4.5 `frontend-winui/Services/TaskApiClient.cs`

```csharp
using System.Net.Http.Json;
using RemoteTaskManager.UI.Models;

namespace RemoteTaskManager.UI.Services;

public class TaskApiClient
{
    private readonly HttpClient _http = new() { BaseAddress = new Uri("http://localhost:8080") };

    public async Task<List<TaskItem>> GetTasksAsync()
        => await _http.GetFromJsonAsync<List<TaskItem>>("/tasks") ?? [];

    public async Task<TaskItem?> AddTaskAsync(string title)
    {
        var res = await _http.PostAsJsonAsync("/tasks", new { title });
        res.EnsureSuccessStatusCode();
        return await res.Content.ReadFromJsonAsync<TaskItem>();
    }

    public async Task<TaskItem?> UpdateTaskAsync(TaskItem task)
    {
        var res = await _http.PutAsJsonAsync($"/tasks/{task.Id}", new { title = task.Title, completed = task.Completed });
        res.EnsureSuccessStatusCode();
        return await res.Content.ReadFromJsonAsync<TaskItem>();
    }

    public async Task DeleteTaskAsync(int id)
    {
        var res = await _http.DeleteAsync($"/tasks/{id}");
        res.EnsureSuccessStatusCode();
    }
}
```

### 4.6 `frontend-winui/MainWindow.xaml`

```xml
<Window
    x:Class="RemoteTaskManager.UI.MainWindow"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    mc:Ignorable="d"
    Title="Remote Task Manager">

    <Grid Padding="16" RowDefinitions="Auto,Auto,*">
        <TextBlock Text="Remote Task Manager" FontSize="24" FontWeight="SemiBold" Margin="0,0,0,12" />

        <StackPanel Grid.Row="1" Orientation="Horizontal" Spacing="8" Margin="0,0,0,12">
            <TextBox x:Name="TitleInput" Width="300" PlaceholderText="Task title" />
            <Button Content="Add" Click="Add_Click" />
            <Button Content="Refresh" Click="Refresh_Click" />
        </StackPanel>

        <ListView Grid.Row="2" ItemsSource="{x:Bind Tasks}">
            <ListView.ItemTemplate>
                <DataTemplate x:DataType="local:TaskItem" xmlns:local="using:RemoteTaskManager.UI.Models">
                    <Grid ColumnDefinitions="Auto,*,Auto" Padding="0,4">
                        <CheckBox IsChecked="{x:Bind Completed, Mode=TwoWay}" Checked="TaskCheck_Changed" Unchecked="TaskCheck_Changed" Tag="{x:Bind}" />
                        <TextBox Grid.Column="1" Text="{x:Bind Title, Mode=TwoWay}" LostFocus="TaskTitle_LostFocus" Tag="{x:Bind}" Margin="8,0" />
                        <Button Grid.Column="2" Content="Delete" Tag="{x:Bind}" Click="Delete_Click" />
                    </Grid>
                </DataTemplate>
            </ListView.ItemTemplate>
        </ListView>
    </Grid>
</Window>
```

### 4.7 `frontend-winui/MainWindow.xaml.cs`

```csharp
using System.Collections.ObjectModel;
using Microsoft.UI.Xaml;
using Microsoft.UI.Xaml.Controls;
using RemoteTaskManager.UI.Models;
using RemoteTaskManager.UI.Services;

namespace RemoteTaskManager.UI;

public sealed partial class MainWindow : Window
{
    private readonly TaskApiClient _api = new();
    public ObservableCollection<TaskItem> Tasks { get; } = [];

    public MainWindow()
    {
        this.InitializeComponent();
        _ = LoadTasksAsync();
    }

    private async Task LoadTasksAsync()
    {
        var items = await _api.GetTasksAsync();
        Tasks.Clear();
        foreach (var t in items) Tasks.Add(t);
    }

    private async void Refresh_Click(object sender, RoutedEventArgs e) => await LoadTasksAsync();

    private async void Add_Click(object sender, RoutedEventArgs e)
    {
        if (string.IsNullOrWhiteSpace(TitleInput.Text)) return;
        var created = await _api.AddTaskAsync(TitleInput.Text.Trim());
        TitleInput.Text = "";
        if (created is not null) Tasks.Add(created);
    }

    private async void TaskCheck_Changed(object sender, RoutedEventArgs e)
    {
        if (sender is CheckBox { Tag: TaskItem task })
            await _api.UpdateTaskAsync(task);
    }

    private async void TaskTitle_LostFocus(object sender, RoutedEventArgs e)
    {
        if (sender is TextBox { Tag: TaskItem task })
            await _api.UpdateTaskAsync(task);
    }

    private async void Delete_Click(object sender, RoutedEventArgs e)
    {
        if (sender is Button { Tag: TaskItem task }) {
            await _api.DeleteTaskAsync(task.Id);
            Tasks.Remove(task);
        }
    }
}
```

---

## 5) Frontend build/run commands

```powershell
cd frontend-winui
# restore + build
 dotnet restore
 dotnet build
# run
 dotnet run
```

---

## 6) Run full system

1. Start backend:
   ```powershell
   cd backend-win32
   .\build\RemoteTaskManagerBackend.exe
   ```
2. Start frontend:
   ```powershell
   cd frontend-winui
   dotnet run
   ```
3. Add/update/delete tasks in UI.
4. Validate persistence in memory via:
   ```powershell
   curl http://localhost:8080/tasks
   ```

---

## 7) Next production improvements

- Use a robust JSON library (`nlohmann/json`) and HTTP framework (`cpp-httplib`/Boost.Beast).
- Add authentication (JWT or Windows auth).
- Add TLS/HTTPS (reverse proxy like IIS/NGINX).
- Add SQLite storage instead of memory.
- Add logging/metrics.
- Package frontend with MSIX.
