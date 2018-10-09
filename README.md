# Avalab.Serilog.Sanitizer [![build status](https://gitlab.avalab.io/Shared/Avalab.Serilog.Sanitizer/badges/master/build.svg)](https://gitlab.avalab.io/Shared/Avalab.Serilog.Sanitizer/tree/master)

## ���� � ����������

���������� ������������� ��� ������������ �����-���� ���������� � �����, ����������� �� ���� Serilog.  
��������� �� ����������� ��������� Serilog, ���������� Sinks.

### �����������

* .NETStandard 2.0  
* Serilog 2.6.0+

### ������� �����

���������� ���������� ����� Avalab NuGet Repository ["https://nuget.avalab.io/api/v2/"](https://nuget.avalab.io/api/v2/), 
�������� �������:

```
Install-Package -Source "https://nuget.avalab.io/api/v2/" Avalab.Serilog.Sanitizer
```

�.�. ���������� �������� ����������� ```ILogEventSink```, �� ����������� �������������� ����� 
```LoggerConfiguration().WriteTo```

```csharp
var logger = new LoggerConfiguration()
                 .WriteTo.Sanitizer(
                     r => { r.PanUnreadable(); r.CvvHidden(); },
                     s => { s.Debug(); s.Console(); }
             ).CreateLogger();
```

� �������� ������� ������������� ��������� ```rules```, ���������� �������� ��������� ```AbstractSanitizingRule[]```. 
��� ��� ���������� ������� ��� ������������ ��������, ���������� ������ �����.  
� �������� ������� ������������� ��������� ```sinks```, ���������� �������� ��������� ```ILogEventSink[]```. ��� ������� **Sinks**,
���� �����, ����� �������������, ����������� ����, ���������� ```LogEvent```.  

��������������, �������������� �������� ```sanitizeException``` ���� ```bool```, �� ��������� ```true```. �������� �� ��, ����� ���������� 
������������ � ��� ����� ```Exception``` ���� �����, ```LogEvent.Exception```.

### ������������ ������ ���������

���������� ����� �������� � ���� 3 ���������� ```AbstractSanitizingRule```:  
* PanUnreadable
* CvvHidden
* RegexHidden

##### PanUnreadable

������� ��������� ����������� ������ ���������, ��������� ����, ����� ������ ����� ���� �� ������ ```*```

� �������� ����������, ���������:  
```csharp
string regularExpression = "[3456]\\d{3}[- ]?\\d{4}[- ]?\\d{4}[- ]?\\d{4}(?:[- ]?\\d{2})?"
string replaceString = "*"
uint startSkipCount = 6
uint endSkipCount = 4
```
, ���:  
```regularExpression``` - ���������� ��������� ��� ������ ��������� ����;  
```replaceString``` - ������ ��� ������ ����� ����;  
```startSkipCount``` - ���������� ������������ ���� � ������;  
```endSkipCount``` - ���������� ������������ ���� � �����.  

##### CvvHidden

������� ��������� ����������� CVV ���� ��������� ����, ����� ������ ���� ���� �� ������ ```*```  

� �������� ����������, ���������:  
```csharp
string regularExpression = "(?i)cvv\"?[ ]?:[ ]?\"?\\d{3}\"?"
string replaceString = "*"
```
, ���:  
```regularExpression``` - ���������� ��������� ��� ������ CVV ���� ��������� ����;  
```replaceString``` - ������ ��� ������ ���� ����.  

##### RegexHidden

������� ��������� ����������� ����� ������, ��������� ����� ���������� ���������, �� ������ ```*```  

� �������� ����������, ���������:  
```csharp
string regularExpression
string replaceExpression
string replaceString = "*"
```
, ���:  
```regularExpression``` - ������������. ���������� ��������� ��� ������ �����;  
```replaceExpression``` - ������������. ���������� ��������� ��� ������ ��������, � �������, ��������� ����� ������ ��������;  
```replaceString``` - ������ ��� ������ ��������.  

### ������������ ����� ���� ������������ appsettings.json

��� ������ ���������� ���������� ���������� ����� NuGet:

```
Install-Package Serilog.Settings.Configuration
```
, ������� ��������� ������ ����� appsettings.json � ���������� ������������ ����� ���:

```csharp
var logger = new LoggerConfiguration()
                 .ReadFrom.Configuration(configuration)
                 .CreateLogger();
```
, ��� ```configuration``` ��������� ```IConfiguration```

������ ������ ������������:  

```json
"Serilog": {
  "Using": [
    "Serilog.Sinks.Trace",
    "Serilog.Sinks.Debug",
    "Avalab.Serilog.Sanitizer" // ����������� ����� ���������� Assembly
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

### ���������� ����������� ������ ���������

���������� ������������ ���������� ����������� ������ �����������.  
��� ����� ���������� ����������� ```AbstractSanitizingRule``` � ���� ������������ �����:  

```csharp
sealed class MyCustomSanitizingRule : AbstractSanitizingRule
{
    public MyCustomSanitizingRule(string param1, int param2)
    {
       // ...
    }

    public string Sanitize(string content)
    {
        // ���� ����������
    }
}
```

� ����� �������� ����� ���������� ����:
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

����� ����� ��������� ������� � ������������, �� ����� ������� Assembly, ��� ����� ��� �������:

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
