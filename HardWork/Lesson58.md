К сожалению последние куски кода являются чисто работой с json Объектами, сущности, которые я писал ранее уже приведены к более функциональному виду 
```cs
.AddStep(GetEntity)
.AddStep(CreateRequest)
.AddTransparentProxyRequest<EmployeeEntity>(PostNewEmpoyeeRequest)
...
```

Материал изучен, в будущем когда появится возможность, обязательно применю его.
