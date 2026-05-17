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

Open Supabase and execute the following SQL scripts in the SQL Editor.

![SQL Editor](images/sql_editor.png)

## 1.1 Inventory Table
```sql
create extension if not exists pgcrypto;

create table inventory (
    id uuid primary key default gen_random_uuid(),
    name text,
    quantity int,
    created_at timestamptz default now()
);
```

## 1.2 Borrowed Table
```sql
create table borrowed (
    borrow_id text primary key,
    inventory_id uuid references inventory(id) on delete cascade,
    user_id bigint references users(id) on delete cascade,
    transaction_date timestamptz default now(),
    quantity int not null
);
```

## 1.3 Return Table
```sql
create table returned (
    return_id text primary key,
    borrow_id text references borrowed(borrow_id) on delete cascade,
    inventory_id uuid references inventory(id) on delete cascade,
    user_id bigint references users(id) on delete cascade,
    return_date timestamptz default now(),
    quantity int not null
);
```

All of these are called Table and they will be used to store transactions. After, that we will add a view:
Once you created the tables, your data maodel should look something like this:

![Data Model](images/data_model.png)

## 1.4 Inventory Availability View
```sql
create or replace view inventory_availability as
with borrow_totals as (
    select 
        inventory_id,
        coalesce(sum(quantity), 0) as total_borrowed
    from borrowed
    group by inventory_id
),
return_totals as (
    select 
        inventory_id,
        coalesce(sum(quantity), 0) as total_returned
    from returned
    group by inventory_id
),
latest_activity as (
    select 
        inventory_id,
        greatest(
            coalesce(max(transaction_date), '1900-01-01'),
            coalesce((select max(return_date) from returned r2 where r2.inventory_id = b.inventory_id), '1900-01-01')
        ) as latest_date,
        case 
            when (select max(transaction_date) from borrowed b2 where b2.inventory_id = b.inventory_id) >=
                 (select max(return_date) from returned r2 where r2.inventory_id = b.inventory_id)
            then 'Borrowed'
            else 'Returned'
        end as latest_type
    from borrowed b
    group by inventory_id
)
select 
    i.id as inventory_id,
    i.name as inventory_name,
    i.quantity as max_quantity,
    (i.quantity 
        - coalesce(bt.total_borrowed, 0) 
        + coalesce(rt.total_returned, 0)
    ) as remaining_quantity,
    la.latest_date as latest_transaction_date,
    la.latest_type as latest_transaction_type
from inventory i
left join borrow_totals bt on bt.inventory_id = i.id
left join return_totals rt on rt.inventory_id = i.id
left join latest_activity la on la.inventory_id = i.id;
```

While this query might look too complex, what is does is it gives you the remaining number of inventoey you have. 

---

# 2. Stored Procedures
---

Stored Procedures act like a code in the data base. We use the to communicate with the database and it does the heavy lifting in inserting records.

## 2.1 Safe Borrow Stored Procedure (Prevents Over Borrowing)
```sql
create or replace function safe_insert_borrowed_item(
    p_inventory_id uuid,
    p_user_id bigint,
    p_quantity int
)
returns text
language plpgsql
as $$
declare
    remaining int;
    last_id text;
    new_number int;
    new_id text;
begin
    -- Get remaining quantity from the view
    select remaining_quantity into remaining
    from inventory_availability
    where inventory_id = p_inventory_id;

    if remaining is null then
        raise exception 'Inventory item does not exist.';
    end if;

    if remaining <= 0 then
        raise exception 'No stock available to borrow.';
    end if;

    if p_quantity > remaining then
        raise exception 'Cannot borrow more than remaining quantity (%).', remaining;
    end if;

    -- Generate BUR-00001 ID
    select borrow_id into last_id
    from borrowed
    order by borrow_id desc
    limit 1;

    if last_id is null then
        new_number := 1;
    else
        new_number := (regexp_replace(last_id, '\D', '', 'g'))::int + 1;
    end if;

    new_id := 'BUR-' || lpad(new_number::text, 5, '0');

    insert into borrowed (borrow_id, inventory_id, user_id, quantity)
    values (new_id, p_inventory_id, p_user_id, p_quantity);

    return new_id;
end;

```

## 2.2 Safe Return Stored Procedure (No Over Returning)
```sql
create or replace function safe_insert_returned_item(
    p_borrow_id text,
    p_inventory_id uuid,
    p_user_id bigint,
    p_quantity int
)
returns text
language plpgsql
as $$
declare
    last_id text;
    new_number int;
    new_id text;
    borrowed_qty int;
    returned_qty int;
begin
    -- Get total borrowed for this borrow_id
    select quantity into borrowed_qty
    from borrowed
    where borrow_id = p_borrow_id;

    if borrowed_qty is null then
        raise exception 'Borrow record does not exist.';
    end if;

    -- Get total returned so far
    select coalesce(sum(quantity), 0) into returned_qty
    from returned
    where borrow_id = p_borrow_id;

    if returned_qty + p_quantity > borrowed_qty then
        raise exception 'Cannot return more than borrowed.';
    end if;

    -- Generate RET-00001 ID
    select return_id into last_id
    from returned
    order by return_id desc
    limit 1;

    if last_id is null then
        new_number := 1;
    else
        new_number := (regexp_replace(last_id, '\D', '', 'g'))::int + 1;
    end if;

    new_id := 'RET-' || lpad(new_number::text, 5, '0');

    insert into returned (return_id, borrow_id, inventory_id, user_id, quantity)
    values (new_id, p_borrow_id, p_inventory_id, p_user_id, p_quantity);

    return new_id;
end;
$$;

```

---

# 3. .NET MAUI
---



Before we proceed, we need to make 4 files for organizational purposes. Create 4 folders:
1. 
2.
3.
4.

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
