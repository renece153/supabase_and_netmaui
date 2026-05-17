# 📘 Supabase SQL + .NET MAUI Integration Guide  
*A complete reference for the Inventory + Borrow/Return System*

---

## 📑 Table of Contents
- [1. Database Schema](#1-database-schema)
- [2. Stored Procedures](#2-stored-procedures)
- [3. .NET MAUI ViewModels](#3-net-maui-viewmodels)
- [4. Borrow Page UI](#4-borrow-page-ui)
- [5. Picker With Right‑Side Arrow](#5-picker-with-right-side-arrow)

---

# 1. Database Schema
---

## 1.1 Inventory Table
```sql
CREATE TABLE inventory (
    id BIGSERIAL PRIMARY KEY,
    inventory_name TEXT NOT NULL,
    max_quantity INT NOT NULL,
    media_type TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

## 1.2 Transactions Table
```sql
CREATE TABLE transactions (
    id BIGSERIAL PRIMARY KEY,
    inventory_id BIGINT REFERENCES inventory(id),
    quantity INT NOT NULL,
    transaction_type TEXT CHECK (transaction_type IN ('borrow', 'return')),
    created_at TIMESTAMP DEFAULT NOW()
);
```

## 1.3 Inventory Availability View
```sql
CREATE VIEW inventory_availability AS
SELECT 
    i.id,
    i.inventory_name,
    i.max_quantity,
    i.media_type,
    i.max_quantity 
        - COALESCE(SUM(CASE WHEN t.transaction_type = 'borrow' THEN t.quantity ELSE 0 END), 0)
        + COALESCE(SUM(CASE WHEN t.transaction_type = 'return' THEN t.quantity ELSE 0 END), 0)
        AS remaining_quantity
FROM inventory i
LEFT JOIN transactions t ON t.inventory_id = i.id
GROUP BY i.id;
```

---

# 2. Stored Procedures
---

## 2.1 Borrow Item
```sql
CREATE OR REPLACE FUNCTION borrow_item(p_inventory_id BIGINT, p_quantity INT)
RETURNS VOID AS $$
BEGIN
    INSERT INTO transactions (inventory_id, quantity, transaction_type)
    VALUES (p_inventory_id, p_quantity, 'borrow');
END;
$$ LANGUAGE plpgsql;
```

## 2.2 Return Item
```sql
CREATE OR REPLACE FUNCTION return_item(p_inventory_id BIGINT, p_quantity INT)
RETURNS VOID AS $$
BEGIN
    INSERT INTO transactions (inventory_id, quantity, transaction_type)
    VALUES (p_inventory_id, p_quantity, 'return');
END;
$$ LANGUAGE plpgsql;
```

---

# 3. .NET MAUI ViewModels
---

## 3.1 AddInventoryViewModel
```csharp
public class AddInventoryViewModel : INotifyPropertyChanged
{
    private readonly InventoryService _service;

    private string _inventoryName;
    private int _maxQuantity;
    private string _selectedMediaType;

    public string InventoryName
    {
        get => _inventoryName;
        set { _inventoryName = value; OnPropertyChanged(nameof(InventoryName)); }
    }

    public int MaxQuantity
    {
        get => _maxQuantity;
        set { _maxQuantity = value; OnPropertyChanged(nameof(MaxQuantity)); }
    }

    public string SelectedMediaType
    {
        get => _selectedMediaType;
        set { _selectedMediaType = value; OnPropertyChanged(nameof(SelectedMediaType)); }
    }

    public ObservableCollection<string> MediaTypes { get; set; }

    public Command AddCommand { get; }
    public Command CloseCommand { get; }

    public AddInventoryViewModel()
    {
        _service = new InventoryService();

        MediaTypes = new ObservableCollection<string>
        {
            "Book", "DVD", "CD", "Game", "Other"
        };

        AddCommand = new Command(async () => await AddInventory());
        CloseCommand = new Command(async () => await Close());
    }

    private async Task AddInventory()
    {
        await _service.AddInventoryAsync(InventoryName, MaxQuantity, SelectedMediaType);
        await Application.Current.MainPage.Navigation.PopModalAsync();
    }

    private async Task Close() =>
        await Application.Current.MainPage.Navigation.PopModalAsync();

    public event PropertyChangedEventHandler PropertyChanged;
    protected void OnPropertyChanged(string name) =>
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
}
```

---

## 3.2 TransactionViewModel
```csharp
public class TransactionViewModel : INotifyPropertyChanged
{
    private readonly InventoryService _service;

    public ObservableCollection<InventoryAvailability> Inventory { get; set; }

    public Command LoadInventoryCommand { get; }
    public Command<InventoryAvailability> OpenBorrowCommand { get; }
    public Command<InventoryAvailability> OpenReturnCommand { get; }
    public Command AddInventoryCommand { get; }

    public TransactionViewModel()
    {
        _service = new InventoryService();
        Inventory = new ObservableCollection<InventoryAvailability>();

        LoadInventoryCommand = new Command(async () => await LoadInventory());
        OpenBorrowCommand = new Command<InventoryAvailability>(OpenBorrowPage);
        OpenReturnCommand = new Command<InventoryAvailability>(OpenReturnPage);
        AddInventoryCommand = new Command(async () => await OpenAddInventoryPage());

        LoadInventoryCommand.Execute(null);
    }

    private async Task LoadInventory()
    {
        var items = await _service.GetInventoryAvailabilityAsync();
        Inventory.Clear();
        foreach (var item in items)
            Inventory.Add(item);
    }

    private async void OpenBorrowPage(InventoryAvailability item)
    {
        var borrowPage = new BorrowPage(item);
        await Application.Current.MainPage.Navigation.PushModalAsync(borrowPage);
        borrowPage.Disappearing += async (s, e) => await LoadInventory();
    }

    private async void OpenReturnPage(InventoryAvailability item)
    {
        var returnPage = new ReturnPage(item);
        await Application.Current.MainPage.Navigation.PushModalAsync(returnPage);
        returnPage.Disappearing += async (s, e) => await LoadInventory();
    }

    private async Task OpenAddInventoryPage()
    {
        var addPage = new AddInventoryPage();
        await Application.Current.MainPage.Navigation.PushModalAsync(addPage);
        addPage.Disappearing += async (s, e) => await LoadInventory();
    }

    public event PropertyChangedEventHandler PropertyChanged;
}
```

---

# 4. Borrow Page UI
---

```xml
<Grid ColumnDefinitions="*, *" ColumnSpacing="10" VerticalOptions="Center">
    <VerticalStackLayout Grid.Column="0" HorizontalOptions="Center">
        <Label Text="Remaining" TextColor="Gray" HorizontalOptions="Center" />
        <Frame BorderColor="LightGray" CornerRadius="6" Padding="0"
               HeightRequest="40" WidthRequest="100" BackgroundColor="White">
            <Label Text="{Binding SelectedItem.remaining_quantity}"
                   FontAttributes="Bold" TextColor="Green"
                   HorizontalOptions="Center" VerticalOptions="Center" />
        </Frame>
    </VerticalStackLayout>

    <VerticalStackLayout Grid.Column="1" HorizontalOptions="Center">
        <Label Text="Borrowing" TextColor="Gray" HorizontalOptions="Center" />
        <Frame BorderColor="LightGray" CornerRadius="6" Padding="0"
               HeightRequest="40" WidthRequest="100" BackgroundColor="White">
            <Entry Placeholder="Enter quantity" Keyboard="Numeric"
                   BackgroundColor="Transparent" TextColor="Black"
                   HorizontalOptions="Center" VerticalOptions="Center"
                   Text="{Binding Quantity}" />
        </Frame>
    </VerticalStackLayout>
</Grid>
```

---

# 5. Picker With Right‑Side Arrow
---

```xml
<Border Stroke="#B0B0B0" StrokeThickness="1" Background="White"
        HeightRequest="38" StrokeShape="RoundRectangle 6">
    <Grid ColumnDefinitions="*, Auto" VerticalOptions="Center">
        <Picker x:Name="MediaPicker"
                ItemsSource="{Binding MediaTypes}"
                SelectedItem="{Binding SelectedMediaType}"
                Background="Transparent"
                TextColor="Black"
                Margin="8,0" />

        <Path Data="M 0 0 L 10 0 L 5 6 Z"
              Fill="White"
              WidthRequest="12"
              HeightRequest="12"
              VerticalOptions="Center"
              Margin="0,0,10,0"
              Grid.Column="1"
              InputTransparent="True" />
    </Grid>
</Border>
```
