[![Avalab.Serilog.Sanitizer](Avalab.Serilog.Sanitizer.png)](https://github.com/NYMEZIDE/Serilog.Sanitizer)

# Avalab.Serilog.Sanitizer [![build status](https://gitlab.avalab.io/Shared/Avalab.Serilog.Sanitizer/badges/master/build.svg)](https://gitlab.avalab.io/Shared/Avalab.Serilog.Sanitizer/tree/master)

## Цели и назначение

Библиотека предназначена для маскирования какой-либо информации в логах, построенных на базе Serilog.  
Построена на стандартном механизме Serilog, называемом Sinks.

### История изменений

[CHANGELOG.md](CHANGELOG.md)

### Зависимости

* .NETStandard 2.0  
* Serilog 2.5.0+

### Быстрый старт

Подключите библиотеку через Avalab NuGet Repository ["https://nuget.avalab.io/api/v2/"](https://nuget.avalab.io/api/v2/), 
выполнив команду:

```
Install-Package -Source "https://nuget.avalab.io/api/v2/" Avalab.Serilog.Sanitizer
```

Т.к. библиотека является реализацией ```ILogEventSink```, то подключение осуществляется через 
```LoggerConfiguration().WriteTo``` :  
```csharp
var logger = new LoggerConfiguration()
                 .WriteTo.Sanitizer(
                     r => { r.PanUnreadable(); r.CvvHidden(); },
                     s => { s.Debug(); s.Console(); }
             ).CreateLogger();
```

В качестве первого обязательного параметра ```rules```, необходимо передать коллекцию ```AbstractSanitizingRule[]```. 
Это так называемые правила для транформации контента, маскировки текста логов.  
В качестве второго обязательного параметра ```sinks```, необходимо передать коллекцию ```ILogEventSink[]```. Это целевые **Sinks**,
куда затем, после трансформации, отправяться логи, экземпляры ```LogEvent```.  

Дополнительный, необязательный параметр ```sanitizeException``` типа ```bool```, по умолчанию ```true```. Отчечает за то, чтобы библиотека 
обрабатывала в том числе ```Exception``` поле логов, ```LogEvent.Exception```.

### Конфигурация правил обработки

Библиотека также включает в себя 3 реализации ```AbstractSanitizingRule```:  
* PanUnreadable
* CvvHidden
* RegexHidden

##### PanUnreadable

Правило позволяет маскировать номера платежных, кредитных карт, путем замены части цифр на символ ```*```

В качестве параметров, принимает:  
```csharp
string regularExpression = "[3456]\\d{3}[- ]?\\d{4}[- ]?\\d{4}[- ]?\\d{4}(?:[- ]?\\d{2})?"
string replaceString = "*"
uint startSkipCount = 6
uint endSkipCount = 4
```
, где:  
```regularExpression``` - регулярное выражение для поиска кредитных карт;  
```replaceString``` - символ для замены части цифр;  
```startSkipCount``` - количество пропускаемых цифр в начале;  
```endSkipCount``` - количество пропускаемых цифр в конце.  

##### CvvHidden

Правило позволяет маскировать CVV коды кредитных карт, путем замены всех цифр на символ ```*```  

В качестве параметров, принимает:  
```csharp
string regularExpression = "(?i)cvv\"?[ ]?:[ ]?\"?\\d{3}\"?"
string replaceString = "*"
```
, где:  
```regularExpression``` - регулярное выражение для поиска CVV кода кредитных карт;  
```replaceString``` - символ или строка для замены всех цифр.  

##### RegexHidden

Правило позволяет маскировать куски текста, найденные через регулярное выражение, на символ ```*```  

В качестве параметров, принимает:  
```csharp
string regularExpression
string replaceExpression
string replaceString = "*"
```
, где:  
```regularExpression``` - обязательный. Регулярное выражение для поиска строк;  
```replaceExpression``` - обязательный. Регулярное выражение для замены символов, в строках, найденных через первый папаметр;  
```replaceString``` - символ или строка для замены.  

### Конфигурация через файл конфигурации appsettings.json

Для начала необходимо подключить библиотеку через NuGet:

```
Install-Package Serilog.Settings.Configuration
```
, которая позволяет читать файлы appsettings.json и передавать конфигурацию через код:

```csharp
var logger = new LoggerConfiguration()
                 .ReadFrom.Configuration(configuration)
                 .CreateLogger();
```
, где ```configuration``` реализует ```IConfiguration```

Пример секции конфигурации:  

```json
"Serilog": {
  "Using": [
    "Serilog.Sinks.Trace",
    "Serilog.Sinks.Debug",
    "Avalab.Serilog.Sanitizer" // обязательно нужно подключить Assembly
  ],
...
"WriteTo": [
  {
    "Name": "Sanitizer",
    "Args": {
      "rules": [
        {
          "Name": "PanUnreadable",
          "Args": {
            "regularExpression": "[3456]\\d{3}[- ]?\\d{4}[- ]?\\d{4}[- ]?\\d{4}(?:[- ]?\\d{2})?",
            "replaceString": "*"
          }
        },
        {
          "Name": "CvvHidden",
          "Args": {
            "regularExpression": "(?i)cvv\"?[ ]?:[ ]?\"?\\d{3}\"?",
            "replaceString": "*"
          }
        }
      ],
      "sinks": [
        {
          "Name": "Trace",
          "Args": {
            "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level}] {Message}{NewLine}{Exception}"
          }
        },
        {
          "Name": "Debug",
          "Args": {
            "outputTemplate": "[{Level}] {Message}{NewLine}{Exception}"
          }
        }
      ]
    }
  }
]
...
```

### Добавление собственных правил санитарии

Библиотека поддерживает добавление собственных правил модификации.  
Для этого необходимо реализовать ```AbstractSanitizingRule``` , один единственный публичный метод ```Sanitize```:  

```csharp
sealed class MyCustomSanitizingRule : AbstractSanitizingRule
{
    public MyCustomSanitizingRule(string param1, int param2)
    {
       // ...
    }

    public string Sanitize(string content)
    {
        // реализация: необходимо вернуть измененную строку
    }
}
```

А также добавить метод расширения вида:
```csharp
public static LoggerConfiguration MyCustomRule(
    this LoggerSinkConfiguration loggerSinkConfiguration,
    string param1 = "value1",
    int param2 = 123)
{
    if (loggerSinkConfiguration == null)
        throw new ArgumentNullException(nameof(loggerSinkConfiguration));

    return loggerSinkConfiguration.Sink(new MyCustomSanitizingRule(param1, param2));
}
```

После можно прописать правило в конфигурацию, не забыв указать Assembly, где лежит это правило:

```json
"Using": [
    "MyCustomRuleAssembly" 
  ],
...
"rules": [
  {
    "Name": "MyCustomRule",
    "Args": {
      "param1": "value1",
      "param2": 123
    }
  }
...
```
