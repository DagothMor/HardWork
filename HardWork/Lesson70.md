# 1 Конфликт ioc

# 1.1 до

    public static IServiceCollection AddPluginServices(this IServiceCollection services)
    {
        services.TryAddScoped<IManager, BusinessManagerOld>();
        services.AddPluginDbContext<BusinessDbContext>();
        return services;
    }
    public static IServiceCollection AddPluginServicesForNewPipeline(this IServiceCollection services)
    {
        services.AddScoped<IManager, BusinessManagerPipeline>();
        services.AddPluginDbContext<BusinessDbContext>();
        return services;
    }

# 1.1 после

public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddPluginServices(this IServiceCollection services)
    {
        services.AddScoped<IManagerOld, ManagerOld>();
        services.AddPluginDbContext<BusinessDbContext>();
        return services;
    }
    public static IServiceCollection AddPluginServicesForNewPipeline(this IServiceCollection services)
    {
        services.AddScoped<IManagerPipeline, ManagerPipeline>();
        services.AddPluginDbContext<BusinessDbContext>();
        return services;
    }
}


# 2 CRUD

Очень удобно создавать CRUD интерфейсы, для дефолтных сущностей можно просто повесить ICRUDable, а для, допустим, карточки пользователя, ICreatable,IEditable,IReadable, т.е без IDeletable, поскольку мы зависимы от закона обязующего нас хранить все данные о пользователе.


# Итоги
По сути почти все интерфейсы на работе являются сборником более мелких атомарных интерфейсов, на проблемы я натыкался только при работе c ioc.
