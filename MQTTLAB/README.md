# README.md

## Описание проекта

Данный проект реализует взаимодействие с MQTT-брокером для публикации и подписки на сообщения. Проект включает в себя клиентскую часть для публикации сообщений и подписчика для получения сообщений. Он использует библиотеку `MQTTnet` для работы с MQTT-протоколом и поддерживает настройку SSL/TLS для безопасного соединения.

---

## Основные компоненты

### 1. MQTT-клиент (`Mqtt`)

Класс `Mqtt` реализует функциональность для подключения к MQTT-брокеру, публикации и подписки на сообщения.

#### Основные моменты:

- **Инициализация MQTT-клиента**:
  ```csharp
  public Mqtt(ILogger logger, MqttConnectionSettings mqttSettings)
  {
      this.logger = logger;
      this.mqttSettings = mqttSettings;
      JsonSerializerOptions = new JsonSerializerOptions()
      {
          PropertyNameCaseInsensitive = true,
          WriteIndented = true,
          PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
          IncludeFields = true,
          DefaultIgnoreCondition = System.Text.Json.Serialization.JsonIgnoreCondition.WhenWritingNull
      };
  }
  ```
  - Конструктор класса принимает настройки подключения (`MqttConnectionSettings`) и настраивает параметры сериализации JSON.

- **Настройка SSL/TLS**:
  ```csharp
  if (mqttSettings.SslProtocol != null)
  {
      SslProtocols sslProtocol;
      if (Enum.TryParse<SslProtocols>(mqttSettings.SslProtocol, out sslProtocol))
      {
          var tlsOptions = new MqttClientTlsOptionsBuilder()
              .UseTls()
              .WithSslProtocols(sslProtocol);
          X509Certificate2 rootCrt = new X509Certificate2("rootCA.crt");

          tlsOptions.WithCertificateValidationHandler((cert) =>
          {
              try
              {
                  if (cert.SslPolicyErrors == SslPolicyErrors.None)
                  {
                      return true;
                  }

                  if (cert.SslPolicyErrors == SslPolicyErrors.RemoteCertificateChainErrors)
                  {
                      cert.Chain.ChainPolicy.RevocationMode = X509RevocationMode.NoCheck;
                      cert.Chain.ChainPolicy.VerificationFlags = X509VerificationFlags.NoFlag;
                      cert.Chain.ChainPolicy.ExtraStore.Add(rootCrt);

                      cert.Chain.Build((X509Certificate2)rootCrt);
                      var res = cert.Chain.ChainElements.Cast<X509ChainElement>().Any(a => a.Certificate.Thumbprint == rootCrt.Thumbprint);
                      return res;
                  }
              }
              catch { }

              return false;
          });

          mqttClientOptionsBuilder.WithTlsOptions(tlsOptions.Build());
      }
  }
  ```
  - Если указан протокол SSL/TLS, настраивается безопасное соединение с использованием сертификата (`rootCA.crt`).

- **Подписка на сообщения**:
  ```csharp
  public async Task Subscribe(string topic, MqttQualityOfServiceLevel qualityOfServiceLevel)
  {
      await managedMqttClient.SubscribeAsync(topic, qualityOfServiceLevel);
  }
  ```
  - Метод подписывает клиента на указанный топик с заданным уровнем качества обслуживания (QoS).

- **Публикация сообщений**:
  ```csharp
  public async Task PublishAsync(string topic, object data, string format = "json")
  {
      byte[] payload;
      switch (format.ToLower())
      {
          case "json":
              var json = JsonSerializer.Serialize(data, JsonSerializerOptions);
              payload = Encoding.UTF8.GetBytes(json);
              break;
          case "text":
              payload = Encoding.UTF8.GetBytes(data.ToString());
              break;
          case "binary":
              payload = data as byte[] ?? throw new ArgumentException("Data must be of type byte[] for binary format.");
              break;
          default:
              throw new ArgumentException($"Unsupported format: {format}");
      }

      var mqttMessage = new MqttApplicationMessageBuilder()
          .WithTopic(topic)
          .WithPayload(payload)
          .WithQualityOfServiceLevel(mqttSettings.QOS)
          .WithRetainFlag()
          .Build();

      if (managedMqttClient == null)
      {
          await BuildAsync();
      }

      await managedMqttClient.EnqueueAsync(mqttMessage);
      logger.LogInformation($"Published message to topic '{topic}' in format '{format}'.");
  }
  ```
  - Метод публикует сообщение в указанный топик. Поддерживает форматы: JSON, текст и бинарные данные.

---

### 2. MQTT-публикатор (`Publisher`)

Программа `Publisher` отвечает за отправку сообщений на MQTT-брокер.

#### Основные моменты:

- **Инициализация конфигурации**:
  ```csharp
  private static IConfiguration InitConfiguration()
  {
      var config = new ConfigurationBuilder()
          .AddJsonFile("appsettings.json")
          .Build();
      return config;
  }
  ```
  - Конфигурация загружается из файла `appsettings.json`.

- **Публикация сообщений**:
  ```csharp
  private static async Task StartPublishAsync(Common.Mqtt mqttSender)
  {
      var topic = configuration.GetValue<string>("Topic");

      Console.WriteLine("Enter your messages (type 'exit' to quit):");

      while (true)
      {
          Console.Write("> ");
          var message = Console.ReadLine();

          if (message?.ToLower() == "exit")
              break;

          await mqttSender.PublishAsync(topic, message);
          Console.WriteLine($"Sent: {message}");
      }
  }
  ```
  - Пользователь вводит сообщения, которые публикуются в указанный топик. Для выхода вводится команда `exit`.

---

### 3. MQTT-подписчик (`Subscriber`)

Программа `Subscriber` отвечает за получение сообщений от MQTT-брокера.

#### Основные моменты:

- **Подписка на топик**:
  ```csharp
  private static async Task SubscribeAsync(Common.Mqtt eventBus)
  {
      var topic = configuration.GetValue<string>("Topic");
      await eventBus.Subscribe(topic, MqttConnectionSettings.QOS);
  }
  ```
  - Подписчик подписывается на указанный топик с заданным уровнем QoS.

- **Обработка сообщений**:
  ```csharp
  private static void Handler(object sender, (string Topic, object Data) obj)
  {
      var (topic, data) = obj;

      string payload = data.ToString();

      try
      {
          var json = JsonSerializer.Deserialize<Dictionary<string, object>>(payload);
          Logger.LogInformation($"Received JSON message on topic '{topic}': {JsonSerializer.Serialize(json)}");
      }
      catch (JsonException)
      {
          Logger.LogInformation($"Received text message on topic '{topic}': {payload}");
      }
      catch (Exception ex)
      {
          Logger.LogError($"Failed to process message on topic '{topic}': {ex.Message}");
      }
  }
  ```
  - Обрабатывает сообщения, пришедшие по топику. Если сообщение в формате JSON, оно десериализуется и логируется. В противном случае сообщение обрабатывается как текст.

---
