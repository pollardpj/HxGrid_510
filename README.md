# Repro for https://github.com/havit/Havit.Blazor/issues/510

Create a new Blazor Web Assembly project, ASP.NET Hosted, targetting net6.0
Add package Havit.Blazor.Components.Web.Bootstrap 3.2.10
Add @using Havit.Blazor.Components.Web.Bootstrap to _Imports.razor

Add class in Client project in Pages folder:

public class GridDataRow
{
    public string Value { get; set; }
}

Add to Program.cs in Client:

```
builder.Services.AddHxServices();
```

Add to Index.razor:

```
@page "/"

<PageTitle>Index</PageTitle>

<HxGrid TItem="GridDataRow" DataProvider="GetGridDataRowData" SelectionEnabled="false">
    <Columns>
        <HxGridColumn ItemTextSelector="@(context => context.Value)">
            <HeaderTemplate>
                <span>The Value</span>
            </HeaderTemplate>
        </HxGridColumn>
    </Columns>
</HxGrid>

@code {

    private async Task<GridDataProviderResult<GridDataRow>> GetGridDataRowData(GridDataProviderRequest<GridDataRow> request)
    {
        var results = new List<GridDataRow>
        {
            new GridDataRow { Value = "One" }
        };

        return await Task.FromResult(request.ApplyTo(results));
    }
}
```

Run the Blazor server project, and all works as expected.

Add docker support to the server project:

```
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["BlazorApp1/Server/BlazorApp1.Server.csproj", "BlazorApp1/Server/"]
COPY ["BlazorApp1/Client/BlazorApp1.Client.csproj", "BlazorApp1/Client/"]
COPY ["BlazorApp1/Shared/BlazorApp1.Shared.csproj", "BlazorApp1/Shared/"]
RUN dotnet restore "BlazorApp1/Server/BlazorApp1.Server.csproj"
COPY . .
WORKDIR "/src/BlazorApp1/Server"
RUN dotnet build "BlazorApp1.Server.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "BlazorApp1.Server.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "BlazorApp1.Server.dll"]
```

Try and build that docker image:

```
BlazorApp1> docker build -f .\BlazorApp1\Server\Dockerfile .
```

And it fails on;

```
error CS0234: The type or namespace name 'GridHeaderCellContext' does not exist in the namespace '__Blazor.Havit.Blazor.Components.Web.Bootstrap' (are you missing an assembly reference?)
```

If you change the build image to:

```
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
```

It builds OK.