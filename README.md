# mcp-server
{
  "name": "weather-mcp",
  "version": "1.0.0",
  "description": "Weather MCP server for Cline using OpenWeatherMap API",
  "main": "build/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node build/index.js",
    "dev": "ts-node src/index.ts"
  },
  "author": "",
  "license": "MIT",
  "dependencies": {
    "@modelcontextprotocol/server": "^0.5.0",
    "axios": "^1.6.7",
    "node-cache": "^5.1.2"
  },
  "devDependencies": {
    "@types/node": "^20.11.5",
    "ts-node": "^10.9.2",
    "typescript": "^5.3.3"
  }
}
{
  "compilerOptions": {
    "target": "es2020",
    "module": "commonjs",
    "lib": ["es2020"],
    "outDir": "./build",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
import {
  Server,
  McpError,
  ErrorCode,
  Tool,
  ToolParam,
  ToolExecuteRequestSchema,
  ListResourcesRequestSchema,
  ReadResourceRequestSchema
} from "@modelcontextprotocol/server";
import axios from "axios";
import NodeCache from "node-cache";

// 日志级别
const LOG_LEVEL = process.env.LOG_LEVEL || "info";
const DEBUG = LOG_LEVEL === "debug";

// 天气API配置
interface WeatherConfig {
  apiKey: string;
  baseUrl: string;
  units: "metric" | "imperial" | "standard";
}

// 缓存配置（秒）
const CACHE_TTL = {
  CURRENT_WEATHER: 30 * 60, // 30分钟
  FORECAST: 60 * 60, // 1小时
};

// 主类
class WeatherMcpServer {
  private config: WeatherConfig;
  private server: Server;
  private cache: NodeCache;

  constructor() {
    console.error("[Setup] Initializing Weather MCP server...");
    
    // 获取API密钥
    const apiKey = process.env.OPENWEATHERMAP_API_KEY;
    if (!apiKey) {
      console.error("[Error] Missing OPENWEATHERMAP_API_KEY environment variable");
      process.exit(1);
    }

    // 初始化配置
    this.config = {
      apiKey,
      baseUrl: "https://api.openweathermap.org/data/2.5",
      units: (process.env.WEATHER_UNITS as any) || "metric"
    };

    // 初始化缓存
    this.cache = new NodeCache({
      stdTTL: CACHE_TTL.CURRENT_WEATHER,
      checkperiod: 120
    });

    // 初始化MCP服务器
    this.server = new Server();
    
    // 注册处理程序
    this.registerHandlers();
    
    console.error("[Setup] Weather MCP server initialized successfully");
  }

  private registerHandlers(): void {
    // 工具执行处理
    this.server.setRequestHandler(ToolExecuteRequestSchema, async (request) => {
      console.error(`[Tool] Executing tool: ${request.params.name}`);
      
      try {
        switch (request.params.name) {
          case "get_current_weather":
            return await this.getCurrentWeather(request.params.params);
          case "get_weather_forecast":
            return await this.getWeatherForecast(request.params.params);
          default:
            throw new McpError(ErrorCode.MethodNotFound, `Unknown tool: ${request.params.name}`);
        }
      } catch (error) {
        console.error(`[Error] Tool execution failed: ${error instanceof Error ? error.message : String(error)}`);
        
        if (error instanceof McpError) {
          throw error;
        }
        
        return {
          content: [{
            type: "text",
            text: `Error: ${error instanceof Error ? error.message : String(error)}`
          }],
          isError: true
        };
      }
    });

    // 工具列表
    this.server.setRequestHandler(ToolExecuteRequestSchema.definitions.toolsRequest, async () => {
      console.error("[Tools] Getting tools list");
      
      return {
        tools: [
          {
            name: "get_current_weather",
            description: "Get the current weather for a location",
            parameters: {
              type: "object",
              properties: {
                location: {
                  type: "string",
                  description: "City name (e.g., 'London'), or city name with country code (e.g., 'London,uk')"
                }
              },
              required: ["location"]
            }
          },
          {
            name: "get_weather_forecast",
            description: "Get a 5-day weather forecast for a location",
            parameters: {
              type: "object",
              properties: {
                location: {
                  type: "string",
                  description: "City name (e.g., 'London'), or city name with country code (e.g., 'London,uk')"
                },
                days: {
                  type: "number",
                  description: "Number of days to forecast (1-5)",
                  default: 3
                }
              },
              required: ["location"]
            }
          }
        ]
      };
    });

    // 资源列表
    this.server.setRequestHandler(ListResourcesRequestSchema, async () => {
      console.error("[Resources] Getting resources list");
      
      return {
        resources: [
          {
            uri: "file:///weather-mcp/readme.md",
            name: "Weather MCP Server README",
            mimeType: "text/markdown"
          }
        ]
      };
    });

    // 资源读取
    this.server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
      console.error(`[Resources] Reading resource: ${request.params.uri}`);
      
      if (request.params.uri === "file:///weather-mcp/readme.md") {
        return {
          contents: [{
            uri: request.params.uri,
            mimeType: "text/markdown",
            text: this.getReadmeContent()
          }]
        };
      }
      
      throw new McpError(ErrorCode.ResourceNotFound, `Resource not found: ${request.params.uri}`);
    });
  }

  private async getCurrentWeather(params: any): Promise<Tool.ExecuteResponse> {
    this.validateLocation(params.location);
    
    const location = params.location;
    const cacheKey = `current:${location}`;
    
    // 检查缓存
    const cachedData = this.cache.get<any>(cacheKey);
    if (cachedData) {
      console.error(`[Cache] Using cached current weather for: ${location}`);
      return cachedData;
    }
    
    try {
      console.error(`[API] Getting current weather for: ${location}`);
      
      const response = await axios.get(`${this.config.baseUrl}/weather`, {
        params: {
          q: location,
          appid: this.config.apiKey,
          units: this.config.units
        }
      });
      
      const data = response.data;
      
      // 格式化数据
      const result = {
        content: [{
          type: "text",
          text: this.formatCurrentWeather(data)
        }]
      };
      
      // 存入缓存
      this.cache.set(cacheKey, result, CACHE_TTL.CURRENT_WEATHER);
      
      return result;
    } catch (error) {
      if (axios.isAxiosError(error) && error.response?.status === 404) {
        throw new McpError(ErrorCode.InvalidParams, `Location not found: ${location}`);
      }
      throw error;
    }
  }

  private async getWeatherForecast(params: any): Promise<Tool.ExecuteResponse> {
    this.validateLocation(params.location);
    
    const location = params.location;
    const days = Math.min(Math.max(parseInt(params.days || "3", 10), 1), 5);
    const cacheKey = `forecast:${location}:${days}`;
    
    // 检查缓存
    const cachedData = this.cache.get<any>(cacheKey);
    if (cachedData) {
      console.error(`[Cache] Using cached forecast for: ${location} (${days} days)`);
      return cachedData;
    }
    
    try {
      console.error(`[API] Getting ${days}-day forecast for: ${location}`);
      
      const response = await axios.get(`${this.config.baseUrl}/forecast`, {
        params: {
          q: location,
          appid: this.config.apiKey,
          units: this.config.units
        }
      });
      
      const data = response.data;
      
      // 格式化数据
      const result = {
        content: [{
          type: "text",
          text: this.formatForecast(data, days)
        }]
      };
      
      // 存入缓存
      this.cache.set(cacheKey, result, CACHE_TTL.FORECAST);
      
      return result;
    } catch (error) {
      if (axios.isAxiosError(error) && error.response?.status === 404) {
        throw new McpError(ErrorCode.InvalidParams, `Location not found: ${location}`);
      }
      throw error;
    }
  }

  private validateLocation(location: unknown): asserts location is string {
    if (typeof location !== "string" || location.trim() === "") {
      throw new McpError(ErrorCode.InvalidParams, "A valid location is required");
    }
  }

  private formatCurrentWeather(data: any): string {
    const temp = Math.round(data.main.temp);
    const feelsLike = Math.round(data.main.feels_like);
    const unit = this.config.units === "imperial" ? "°F" : "°C";
    const windUnit = this.config.units === "imperial" ? "mph" : "m/s";
    
    const weatherDescription = data.weather[0].description;
    const weatherEmoji = this.getWeatherEmoji(data.weather[0].id);
    
    return `# Current Weather in ${data.name}, ${data.sys.country} ${weatherEmoji}\n\n` +
           `**Temperature:** ${temp}${unit} (Feels like: ${feelsLike}${unit})\n` +
           `**Condition:** ${this.capitalizeFirst(weatherDescription)}\n` +
           `**Humidity:** ${data.main.humidity}%\n` +
           `**Wind:** ${data.wind.speed} ${windUnit} from ${this.getWindDirection(data.wind.deg)}\n` +
           `**Visibility:** ${(data.visibility / 1000).toFixed(1)} km\n` +
           `**Pressure:** ${data.main.pressure} hPa\n\n` +
           `*Last updated: ${new Date(data.dt * 1000).toLocaleString()}*`;
  }

  private formatForecast(data: any, days: number): string {
    const city = data.city.name;
    const country = data.city.country;
    const unit = this.config.units === "imperial" ? "°F" : "°C";
    
    // 按天分组
    const dailyForecasts: any = {};
    
    // 处理5天/3小时预报数据
    for (const item of data.list) {
      const date = new Date(item.dt * 1000);
      const day = date.toISOString().split('T')[0];
      
      if (!dailyForecasts[day]) {
        dailyForecasts[day] = {
          date: date,
          minTemp: Infinity,
          maxTemp: -Infinity,
          conditions: [],
          conditionIds: [],
          rainfall: 0,
          windSpeed: 0,
          entries: 0
        };
      }
      
      const forecast = dailyForecasts[day];
      forecast.entries++;
      
      // 更新温度
      forecast.minTemp = Math.min(forecast.minTemp, item.main.temp_min);
      forecast.maxTemp = Math.max(forecast.maxTemp, item.main.temp_max);
      
      // 更新天气状况
      forecast.conditions.push(item.weather[0].description);
      forecast.conditionIds.push(item.weather[0].id);
      
      // 统计降水量
      if (item.rain && item.rain['3h']) {
        forecast.rainfall += item.rain['3h'];
      }
      
      // 统计风速
      forecast.windSpeed += item.wind.speed;
    }
    
    // 选择最靠前的几天
    const forecastDays = Object.keys(dailyForecasts).sort().slice(0, days);
    
    // 构建结果字符串
    let result = `# ${days}-Day Weather Forecast for ${city}, ${country}\n\n`;
    
    for (const day of forecastDays) {
      const forecast = dailyForecasts[day];
      
      // 计算每天的平均值
      const avgWindSpeed = forecast.windSpeed / forecast.entries;
      
      // 找出最常见的天气状况
      const conditionCounts = forecast.conditionIds.reduce((acc: any, id: number) => {
        acc[id] = (acc[id] || 0) + 1;
        return acc;
      }, {});
      
      const mostCommonConditionId = Object.entries(conditionCounts)
        .sort((a: any, b: any) => b[1] - a[1])[0][0];
      
      const conditionIndex = forecast.conditionIds.indexOf(Number(mostCommonConditionId));
      const mostCommonCondition = forecast.conditions[conditionIndex];
      const weatherEmoji = this.getWeatherEmoji(Number(mostCommonConditionId));
      
      // 格式化日期
      const dateOptions: Intl.DateTimeFormatOptions = { weekday: 'long', month: 'short', day: 'numeric' };
      const formattedDate = forecast.date.toLocaleDateString('en-US', dateOptions);
      
      result += `## ${formattedDate} ${weatherEmoji}\n\n` +
                `**Temperature:** ${Math.round(forecast.minTemp)}${unit} to ${Math.round(forecast.maxTemp)}${unit}\n` +
                `**Conditions:** ${this.capitalizeFirst(mostCommonCondition)}\n`;
      
      if (forecast.rainfall > 0) {
        result += `**Precipitation:** ${forecast.rainfall.toFixed(1)} mm\n`;
      }
      
      result += `**Wind:** ${avgWindSpeed.toFixed(1)} ${this.config.units === "imperial" ? "mph" : "m/s"}\n\n`;
    }
    
    return result;
  }

  private getWeatherEmoji(weatherId: number): string {
    // 根据OpenWeatherMap的天气ID选择表情
    if (weatherId >= 200 && weatherId < 300) return "⛈️"; // 雷暴
    if (weatherId >= 300 && weatherId < 400) return "🌧️"; // 小雨
    if (weatherId >= 500 && weatherId < 600) return "🌧️"; // 雨
    if (weatherId >= 600 && weatherId < 700) return "❄️"; // 雪
    if (weatherId >= 700 && weatherId < 800) return "🌫️"; // 雾
    if (weatherId === 800) return "☀️"; // 晴天
    if (weatherId > 800 && weatherId < 900) return "☁️"; // 多云
    return "🌡️"; // 默认
  }

  private getWindDirection(degrees: number): string {
    const directions = ['N', 'NE', 'E', 'SE', 'S', 'SW', 'W', 'NW'];
    const index = Math.round(degrees / 45) % 8;
    return directions[index];
  }

  private capitalizeFirst(str: string): string {
    return str.charAt(0).toUpperCase() + str.slice(1);
  }

  private getReadmeContent(): string {
    return `# Weather MCP Server

A Cline MCP server that provides real-time weather data using the OpenWeatherMap API.

## Features

- Get current weather for any location worldwide
- Get 5-day weather forecasts with detailed information
- Support for metric and imperial units
- Efficient caching to minimize API calls
- Beautifully formatted markdown responses

## Setup

1. Sign up for a free API key at [OpenWeatherMap](https://openweathermap.org/api)
2. Install the MCP server:

   \`\`\`json
   {
     "mcpServers": {
       "weather-mcp": {
         "command": "node",
         "args": [
           "/path/to/weather-mcp/build/index.js"
         ],
         "env": {
           "OPENWEATHERMAP_API_KEY": "YOUR_API_KEY",
           "WEATHER_UNITS": "metric"
         },
         "disabled": false,
         "autoApprove": []
       }
     }
   }
   \`\`\`

3. Set \`WEATHER_UNITS\` to "metric" (°C, m/s) or "imperial" (°F, mph)

## Available Tools

### get_current_weather

Gets the current weather conditions for a specified location.

**Parameters:**
- \`location\`: City name (e.g., 'London'), or city name with country code (e.g., 'London,uk')

### get_weather_forecast

Gets a 5-day weather forecast for a specified location.

**Parameters:**
- \`location\`: City name (e.g., 'London'), or city name with country code (e.g., 'London,uk')
- \`days\`: Number of days to forecast (1-5, default: 3)

## Example Output

### Current Weather

\`\`\`
# Current Weather in London, GB ☁️

**Temperature:** 12°C (Feels like: 10°C)
**Condition:** Overcast clouds
**Humidity:** 76%
**Wind:** 4.12 m/s from SW
**Visibility:** 10.0 km
**Pressure:** 1013 hPa

*Last updated: 1/15/2024, 2:30:00 PM*
\`\`\`

### Weather Forecast

\`\`\`
# 3-Day Weather Forecast for London, GB

## Monday, Jan 15 🌧️

**Temperature:** 10°C to 13°C
**Conditions:** Light rain
**Precipitation:** 2.4 mm
**Wind:** 5.2 m/s

## Tuesday, Jan 16 ☁️

**Temperature:** 9°C to 12°C
**Conditions:** Cloudy
**Wind:** 4.8 m/s

## Wednesday, Jan 17 ☀️

**Temperature:** 8°C to 14°C
**Conditions:** Clear sky
**Wind:** 3.5 m/s
\`\`\`

## License

MIT
`;
  }

  public start(): void {
    this.server.start();
    console.error("[Server] Weather MCP server started successfully");
  }
}

// 启动服务器
const server = new WeatherMcpServer();
server.start();
# MCP Server Development Protocol

⚠️ CRITICAL: DO NOT USE attempt_completion BEFORE TESTING ⚠️

## Step 1: Planning (PLAN MODE)
- What problem does this tool solve?
- What API/service will it use?
- What are the authentication requirements?
  □ Standard API key
  □ OAuth (requires separate setup script)
  □ Other credentials

## Step 2: Implementation (ACT MODE)
1. Bootstrap
   - For web services, JavaScript integration, or Node.js environments:
     ```bash
     npx @modelcontextprotocol/create-server my-server
     cd my-server
     npm install
     ```
   - For data science, ML workflows, or Python environments:
     ```bash
     pip install mcp
     # Or with uv (recommended)
     uv add "mcp[cli]"
     ```

2. Core Implementation
   - Use MCP SDK
   - Implement comprehensive logging
     - TypeScript (for web/JS projects):
       ```typescript
       console.error('[Setup] Initializing server...');
       console.error('[API] Request to endpoint:', endpoint);
       console.error('[Error] Failed with:', error);
       ```
     - Python (for data science/ML projects):
       ```python
       import logging
       logging.error('[Setup] Initializing server...')
       logging.error(f'[API] Request to endpoint: {endpoint}')
       logging.error(f'[Error] Failed with: {str(error)}')
       ```
   - Add type definitions
   - Handle errors with context
   - Implement rate limiting if needed

3. Configuration
   - Get credentials from user if needed
   - Add to MCP settings:
     - For TypeScript projects:
       ```json
       {
         "mcpServers": {
           "my-server": {
             "command": "node",
             "args": ["path/to/build/index.js"],
             "env": {
               "API_KEY": "key"
             },
             "disabled": false,
             "autoApprove": []
           }
         }
       }
       ```
     - For Python projects:
       ```bash
       # Directly with command line
       mcp install server.py -v API_KEY=key
       
       # Or in settings.json
       {
         "mcpServers": {
           "my-server": {
             "command": "python",
             "args": ["server.py"],
             "env": {
               "API_KEY": "key"
             },
             "disabled": false,
             "autoApprove": []
           }
         }
       }
       ```

## Step 3: Testing (BLOCKER ⛔️)

<thinking>
BEFORE using attempt_completion, I MUST verify:
□ Have I tested EVERY tool?
□ Have I confirmed success from the user for each test?
□ Have I documented the test results?

If ANY answer is "no", I MUST NOT use attempt_completion.
</thinking>

1. Test Each Tool (REQUIRED)
   □ Test each tool with valid inputs
   □ Verify output format is correct
   ⚠️ DO NOT PROCEED UNTIL ALL TOOLS TESTED

## Step 4: Completion
❗ STOP AND VERIFY:
□ Every tool has been tested with valid inputs
□ Output format is correct for each tool

Only after ALL tools have been tested can attempt_completion be used.

## Key Requirements
- ✓ Must use MCP SDK
- ✓ Must have comprehensive logging
- ✓ Must test each tool individually
- ✓ Must handle errors gracefully
- ⛔️ NEVER skip testing before completion

  # Weather MCP Server

A Cline MCP server that provides real-time weather data using the OpenWeatherMap API.

## Features

- Get current weather for any location worldwide
- Get 5-day weather forecasts with detailed information
- Support for metric and imperial units
- Efficient caching to minimize API calls
- Beautifully formatted markdown responses

## Setup

1. Sign up for a free API key at [OpenWeatherMap](https://openweathermap.org/api)
2. Clone this repository:

   ```bash
   git clone https://github.com/yourusername/weather-mcp.git
   cd weather-mcp
   npm install
   npm run build
   ```

3. Add the MCP server to your Cline settings:

   ```json
   {
     "mcpServers": {
       "weather-mcp": {
         "command": "node",
         "args": [
           "/path/to/weather-mcp/build/index.js"
         ],
         "env": {
           "OPENWEATHERMAP_API_KEY": "YOUR_API_KEY",
           "WEATHER_UNITS": "metric"
         },
         "disabled": false,
         "autoApprove": []
       }
     }
   }
   ```

4. Set `WEATHER_UNITS` to "metric" (°C, m/s) or "imperial" (°F, mph)

## Available Tools

### get_current_weather

Gets the current weather conditions for a specified location.

**Parameters:**
- `location`: City name (e.g., 'London'), or city name with country code (e.g., 'London,uk')

### get_weather_forecast

Gets a 5-day weather forecast for a specified location.

**Parameters:**
- `location`: City name (e.g., 'London'), or city name with country code (e.g., 'London,uk')
- `days`: Number of days to forecast (1-5, default: 3)

## Development

1. Install dependencies:
   ```bash
   npm install
   ```

2. Run in development mode:
   ```bash
   npm run dev
   ```

3. Build for production:
   ```bash
   npm run build
   ```

## License

MIT
