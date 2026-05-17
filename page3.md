# 📄 Creating Transactions

In this section, we will create the **Transaction Page** that represent the data structures used throughout the application.  
Models define the shape of your data and are used by services, viewmodels, and views.

## 📑 Table of Contents
- [1. Transaction Model](#Transaction-Models)
- [2. Transaction Services](#Transaction-Services)
- [3. Transaction View Models](#Transaction-ViewModels)
- [4. Transaction View](#Transaction-Views)
- [5. Navigation](#Edit-Navigation)

---
## 📁 Folder Structure Reminder

Your project should now look like this:

```
/Models
/Services
/ViewModels
/Views
```

Place all corresponding files in to the appropriate folders.
---

# Transaction Models
## 📁 Models

### transaction.cs
```csharp
public class BorrowedItem
{
    public string borrow_id { get; set; }
    public long user_id { get; set; }
    public int quantity { get; set; }
}

public class ReturnedItem
{
    public string return_id { get; set; }
    public string borrow_id { get; set; }

}
```

### InventoryAvailability.cs
```csharp
public class InventoryAvailability
{
    public Guid inventory_id { get; set; }
    public string inventory_name { get; set; }
    public int max_quantity { get; set; }
    public int remaining_quantity { get; set; }
    public DateTime? latest_transaction_date { get; set; }
    public string latest_transaction_type { get; set; }
}
```
---

### BorrowedUser.cs
```csharp
public class BorrowedUser
{
    public string borrow_id { get; set; }
    public long user_id { get; set; }
    public string user_name { get; set; }
    public int quantity { get; set; }
}
```


# Transaction Services

## 📁 Services

### UserService.cs
```csharp
public class UserService
{
    private readonly HttpClient _client;

    public UserService()
    {
        _client = new HttpClient();
        _client.DefaultRequestHeaders.Add("apikey", Constants.SupabaseKey);
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", Constants.SupabaseKey);
    }

    public async Task<List<user>> GetUsersAsync()
    {
        var url = $"{Constants.SupabaseUrl}/rest/v1/users?select=id,name";

        var response = await _client.GetAsync(url);
        var json = await response.Content.ReadAsStringAsync();

        return JsonSerializer.Deserialize<List<user>>(json);
    }
}

public class BorrowedUserService
{
    private readonly HttpClient _client;

    public BorrowedUserService()
    {
        _client = new HttpClient();
        _client.DefaultRequestHeaders.Add("apikey", Constants.SupabaseKey);
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", Constants.SupabaseKey);
    }

    public async Task<List<BorrowedUser>> GetBorrowedUsersAsync(Guid inventoryId)
    {
        // Fetch borrowed items
        var borrowedUrl = $"{Constants.SupabaseUrl}/rest/v1/borrowed?select=borrow_id,user_id,quantity&inventory_id=eq.{inventoryId}";
        var borrowedResponse = await _client.GetAsync(borrowedUrl);
        var borrowedJson = await borrowedResponse.Content.ReadAsStringAsync();
        var borrowedItems = JsonSerializer.Deserialize<List<BorrowedItem>>(borrowedJson);

        // Fetch users
        var usersUrl = $"{Constants.SupabaseUrl}/rest/v1/users?select=id,name";
        var usersResponse = await _client.GetAsync(usersUrl);
        var usersJson = await usersResponse.Content.ReadAsStringAsync();
        var users = JsonSerializer.Deserialize<List<user>>(usersJson);

        // Fetch Returns
        var returnedUrl = $"{Constants.SupabaseUrl}/rest/v1/returned?select=return_id,borrow_id";
        var returnedResponse = await _client.GetAsync(returnedUrl);
        var returnedJson = await returnedResponse.Content.ReadAsStringAsync();
        var returnedItems = JsonSerializer.Deserialize<List<ReturnedItem>>(returnedJson);

        // Filter borrowed items that have NOT been returned
        var activeBorrows = borrowedItems
            .Where(b => !returnedItems.Any(r => r.borrow_id == b.borrow_id))
            .ToList();


        // Join in .NET
        var result = (from b in activeBorrows
                      join u in users on b.user_id equals u.id
                      select new BorrowedUser
                      {
                          borrow_id = b.borrow_id,
                          user_id = u.id,
                          user_name = u.name,
                          quantity = b.quantity
                      }).ToList();

        return result;
    }
}
```

### InventoryService.cs
```csharp
public class InventoryService
{
    private readonly HttpClient _client;

    public InventoryService()
    {
        _client = new HttpClient();
        _client.DefaultRequestHeaders.Add("apikey", Constants.SupabaseKey);
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", Constants.SupabaseKey);
    }

    // ---------------------------------------------------------
    // GET INVENTORY AVAILABILITY VIEW
    // ---------------------------------------------------------
    public async Task<List<InventoryAvailability>> GetInventoryAvailabilityAsync()
    {
        var url = $"{Constants.SupabaseUrl}/rest/v1/inventory_availability?select=*";

        var response = await _client.GetAsync(url);
        var json = await response.Content.ReadAsStringAsync();

        return JsonSerializer.Deserialize<List<InventoryAvailability>>(json);
    }

    // ---------------------------------------------------------
    // ADD INVENTORY
    // ---------------------------------------------------------
    public async Task AddInventoryAsync(string name, int quantity, string mediaType)
    {
        var url = $"{Constants.SupabaseUrl}/rest/v1/inventory";

        var body = new
        {
            name = name,
            quantity = quantity,
            media_type = mediaType
        };

        var content = new StringContent(JsonSerializer.Serialize(body), Encoding.UTF8, "application/json");

        var response = await _client.PostAsync(url, content);

        if (!response.IsSuccessStatusCode)
        {
            var err = await response.Content.ReadAsStringAsync();
            throw new Exception($"Supabase error: {err}");
        }
    }

    // ---------------------------------------------------------
    // BORROW ITEM (RPC)
    // ---------------------------------------------------------
    public async Task BorrowAsync(Guid inventoryId, long userId, int qty)
    {
        var url = $"{Constants.SupabaseUrl}/rest/v1/rpc/safe_insert_borrowed_item";

        var body = new
        {
            p_inventory_id = inventoryId,
            p_user_id = userId,
            p_quantity = qty
        };

        var content = new StringContent(JsonSerializer.Serialize(body), Encoding.UTF8, "application/json");

        var response = await _client.PostAsync(url, content);

        if (!response.IsSuccessStatusCode)
        {
            var err = await response.Content.ReadAsStringAsync();
            throw new Exception($"Borrow failed: {err}");
        }
    }

    // ---------------------------------------------------------
    // RETURN ITEM (RPC)
    // ---------------------------------------------------------
    public async Task ReturnAsync(string borrowId, Guid inventoryId, long userId, int qty)
    {
        var url = $"{Constants.SupabaseUrl}/rest/v1/rpc/safe_insert_returned_item";

        var body = new
        {
            p_borrow_id = borrowId,
            p_inventory_id = inventoryId,
            p_user_id = userId,
            p_quantity = qty
        };

        var content = new StringContent(JsonSerializer.Serialize(body), Encoding.UTF8, "application/json");

        var response = await _client.PostAsync(url, content);

        if (!response.IsSuccessStatusCode)
        {
            var err = await response.Content.ReadAsStringAsync();
            throw new Exception($"Return failed: {err}");
        }
    }

    // ---------------------------------------------------------
    // UPDATE INVENTORY
    // ---------------------------------------------------------
    public async Task UpdateInventoryAsync(Guid id, object body)
    {
        var url = $"{Constants.SupabaseUrl}/rest/v1/inventory?id=eq.{id}";

        var content = new StringContent(JsonSerializer.Serialize(body), Encoding.UTF8, "application/json");

        var response = await _client.PatchAsync(url, content);

        if (!response.IsSuccessStatusCode)
        {
            var err = await response.Content.ReadAsStringAsync();
            throw new Exception($"Update failed: {err}");
        }
    }

    // ---------------------------------------------------------
    // DELETE INVENTORY
    // ---------------------------------------------------------
    public async Task DeleteInventoryAsync(Guid id)
    {
        var url = $"{Constants.SupabaseUrl}/rest/v1/inventory?id=eq.{id}";

        var response = await _client.DeleteAsync(url);

        if (!response.IsSuccessStatusCode)
        {
            var err = await response.Content.ReadAsStringAsync();
            throw new Exception($"Delete failed: {err}");
        }
    }
}
```
---

# Transaction ViewModels

## 📁 ViewModels

## AddInventoryViewModel.cs
```csharp
public class AddInventoryViewModel : INotifyPropertyChanged
{
    private readonly InventoryService _service;

    public string InventoryName { get; set; }
    public int MaxQuantity { get; set; }
    public string MediaType { get; set; }

    public string SelectedMediaType { get; set; }

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
        if (string.IsNullOrWhiteSpace(InventoryName))
        {
            await Application.Current.MainPage.DisplayAlert("Error", "Inventory name is required.", "OK");
            return;
        }

        if (MaxQuantity <= 0)
        {
            await Application.Current.MainPage.DisplayAlert("Error", "Quantity must be greater than 0.", "OK");
            return;
        }

        try
        {
            await _service.AddInventoryAsync(InventoryName, MaxQuantity, SelectedMediaType);
            await Application.Current.MainPage.DisplayAlert("Success", "Inventory item added!", "OK");
            await Close();
        }
        catch (Exception ex)
        {
            await Application.Current.MainPage.DisplayAlert("Error", ex.Message, "OK");
        }
    }

    private async Task Close()
    {
        await Application.Current.MainPage.Navigation.PopModalAsync();
    }

    public event PropertyChangedEventHandler PropertyChanged;
}
```
---

## BorrowPageViewModel.cs
```csharp
public class BorrowPageViewModel : INotifyPropertyChanged
{
    private readonly InventoryService _inventoryService;
    private readonly UserService _userService;

    public InventoryAvailability SelectedItem { get; set; }

    // -----------------------------
    // Quantity
    // -----------------------------
    private int _quantity;
    public int Quantity
    {
        get => _quantity;
        set
        {
            _quantity = value;
            OnPropertyChanged(nameof(Quantity));
        }
    }

    private bool _isSearching;
    public bool IsSearching
    {
        get => _isSearching;
        set
        {
            _isSearching = value;
            OnPropertyChanged(nameof(IsSearching));
        }
    }

    // -----------------------------
    // Users (Full List)
    // -----------------------------
    private ObservableCollection<user> _users;
    public ObservableCollection<user> Users
    {
        get => _users;
        set
        {
            _users = value;
            OnPropertyChanged(nameof(Users));
        }
    }

    // -----------------------------
    // Filtered Users (Search Results)
    // -----------------------------
    private ObservableCollection<user> _filteredUsers;
    public ObservableCollection<user> FilteredUsers
    {
        get => _filteredUsers;
        set
        {
            _filteredUsers = value;
            OnPropertyChanged(nameof(FilteredUsers));
        }
    }

    // -----------------------------
    // Selected User
    // -----------------------------
    private user _selectedUser;
    public user SelectedUser
    {
        get => _selectedUser;
        set
        {
            _selectedUser = value;
            OnPropertyChanged(nameof(SelectedUser));

            if (value != null)
            {
                SearchText = value.name; // show selected name in search bar
                IsSearching = false;     // hide the list
            }
        }
    }

    // -----------------------------
    // Search Text
    // -----------------------------
    private string _searchText;
    public string SearchText
    {
        get => _searchText;
        set
        {
            _searchText = value;
            OnPropertyChanged(nameof(SearchText));
            ApplyFilter();
        }
    }

    // -----------------------------
    // Commands
    // -----------------------------
    public Command ConfirmBorrowCommand { get; }
    public Command CloseCommand { get; }

    // -----------------------------
    // Constructor
    // -----------------------------
    public BorrowPageViewModel(InventoryAvailability item)
    {
        SelectedItem = item;

        _inventoryService = new InventoryService();
        _userService = new UserService();

        ConfirmBorrowCommand = new Command(async () => await ConfirmBorrow());
        CloseCommand = new Command(async () => await Close());

        _ = LoadUsers();
    }

    // -----------------------------
    // Load Users from Supabase
    // -----------------------------
    private async Task LoadUsers()
    {
        var users = await _userService.GetUsersAsync();

        Users = new ObservableCollection<user>(users);
        FilteredUsers = new ObservableCollection<user>(users);
    }

    // -----------------------------
    // Apply Search Filter
    // -----------------------------
    private void ApplyFilter()
    {
        if (string.IsNullOrWhiteSpace(SearchText))
        {
            IsSearching = false;
            FilteredUsers = new ObservableCollection<user>();
            return;
        }

        IsSearching = true;

        var filtered = Users
            .Where(u => u.name.Contains(SearchText, StringComparison.OrdinalIgnoreCase))
            .ToList();

        FilteredUsers = new ObservableCollection<user>(filtered);
    }

    // -----------------------------
    // Confirm Borrow
    // -----------------------------
    private async Task ConfirmBorrow()
    {
        if (Quantity <= 0)
        {
            await App.Current.MainPage.DisplayAlert("Error", "Enter a valid quantity.", "OK");
            return;
        }

        if (Quantity > SelectedItem.remaining_quantity)
        {
            await App.Current.MainPage.DisplayAlert("Error",
                $"Max allowed: {SelectedItem.remaining_quantity}",
                "OK");
            return;
        }

        if (SelectedUser == null)
        {
            await App.Current.MainPage.DisplayAlert("Error", "Please select a borrower.", "OK");
            return;
        }

        try
        {
            await _inventoryService.BorrowAsync(
                SelectedItem.inventory_id,
                SelectedUser.id,
                Quantity
            );

            await App.Current.MainPage.DisplayAlert(
                "Success",
                "Borrowed successfully!",
                "OK"
            );

            await Close();
        }
        catch (Exception ex)
        {
            await App.Current.MainPage.DisplayAlert("Error", ex.Message, "OK");
        }
    }

    // -----------------------------
    // Close Modal
    // -----------------------------
    private async Task Close()
    {
        await App.Current.MainPage.Navigation.PopModalAsync();
    }

    // -----------------------------
    // PropertyChanged Helper
    // -----------------------------
    public event PropertyChangedEventHandler PropertyChanged;
    private void OnPropertyChanged(string name)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
    }
}
```
---

## ReturnPageViewModel.cs
```csharp
public class ReturnPageViewModel : INotifyPropertyChanged
{
    private readonly InventoryService _inventoryService;
    private readonly BorrowedUserService _borrowedUserService;

    public InventoryAvailability SelectedItem { get; set; }

    public bool AllowPartialReturns { get; set; } = false; // toggle if needed

    private int _quantity;
    public int Quantity
    {
        get => _quantity;
        set
        {
            _quantity = value;
            OnPropertyChanged(nameof(Quantity));
        }
    }

    private bool _isSearching;
    public bool IsSearching
    {
        get => _isSearching;
        set
        {
            _isSearching = value;
            OnPropertyChanged(nameof(IsSearching));
        }
    }

    private ObservableCollection<BorrowedUser> _borrowedUsers;
    public ObservableCollection<BorrowedUser> BorrowedUsers
    {
        get => _borrowedUsers;
        set
        {
            _borrowedUsers = value;
            OnPropertyChanged(nameof(BorrowedUsers));
        }
    }

    private ObservableCollection<BorrowedUser> _filteredBorrowedUsers;
    public ObservableCollection<BorrowedUser> FilteredBorrowedUsers
    {
        get => _filteredBorrowedUsers;
        set
        {
            _filteredBorrowedUsers = value;
            OnPropertyChanged(nameof(FilteredBorrowedUsers));
        }
    }

    private BorrowedUser _selectedBorrowedUser;
    public BorrowedUser SelectedBorrowedUser
    {
        get => _selectedBorrowedUser;
        set
        {
            _selectedBorrowedUser = value;
            OnPropertyChanged(nameof(SelectedBorrowedUser));

            if (value != null)
            {
                SearchText = value.user_name;
                IsSearching = false;

                // Auto-fill quantity
                Quantity = value.quantity;
            }
        }
    }

    private string _searchText;
    public string SearchText
    {
        get => _searchText;
        set
        {
            _searchText = value;
            OnPropertyChanged(nameof(SearchText));
            ApplyFilter();
        }
    }

    public Command ConfirmReturnCommand { get; }
    public Command CloseCommand { get; }

    public ReturnPageViewModel(InventoryAvailability item)
    {
        SelectedItem = item;

        _inventoryService = new InventoryService();
        _borrowedUserService = new BorrowedUserService();

        ConfirmReturnCommand = new Command(async () => await ConfirmReturn());
        CloseCommand = new Command(async () => await Close());

        _ = LoadBorrowedUsers();
    }

    private async Task LoadBorrowedUsers()
    {
        var users = await _borrowedUserService.GetBorrowedUsersAsync(SelectedItem.inventory_id);
        BorrowedUsers = new ObservableCollection<BorrowedUser>(users);
        FilteredBorrowedUsers = new ObservableCollection<BorrowedUser>(users);
    }

    private void ApplyFilter()
    {
        if (string.IsNullOrWhiteSpace(SearchText))
        {
            IsSearching = false;
            FilteredBorrowedUsers = new ObservableCollection<BorrowedUser>();
            return;
        }

        IsSearching = true;

        var filtered = BorrowedUsers
            .Where(u => u.user_name.Contains(SearchText, StringComparison.OrdinalIgnoreCase))
            .ToList();

        FilteredBorrowedUsers = new ObservableCollection<BorrowedUser>(filtered);
    }

    private async Task ConfirmReturn()
    {
        if (SelectedBorrowedUser == null)
        {
            await App.Current.MainPage.DisplayAlert("Error", "Please select a borrower.", "OK");
            return;
        }

        if (!AllowPartialReturns)
        {
            Quantity = SelectedBorrowedUser.quantity;
        }

        if (Quantity <= 0 || Quantity > SelectedBorrowedUser.quantity)
        {
            await App.Current.MainPage.DisplayAlert("Error", "Invalid return quantity.", "OK");
            return;
        }

        try
        {
            await _inventoryService.ReturnAsync(
                SelectedBorrowedUser.borrow_id,
                SelectedItem.inventory_id,
                SelectedBorrowedUser.user_id,
                Quantity
            );

            await App.Current.MainPage.DisplayAlert("Success", "Returned successfully!", "OK");
            await Close();
        }
        catch (Exception ex)
        {
            await App.Current.MainPage.DisplayAlert("Error", ex.Message, "OK");
        }
    }

    private async Task Close()
    {
        await App.Current.MainPage.Navigation.PopModalAsync();
    }

    public event PropertyChangedEventHandler PropertyChanged;
    private void OnPropertyChanged(string name)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
    }
}
```

## TransactionViewModel.cs
```csharp
public class TransactionViewModel : INotifyPropertyChanged
{
    private readonly InventoryService _service;

    public ObservableCollection<InventoryAvailability> Inventory { get; set; }

    public Command LoadInventoryCommand { get; }
    public Command<InventoryAvailability> OpenBorrowCommand { get; }
    public Command<InventoryAvailability> OpenReturnCommand { get; }

    // ⭐ ADD THIS
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

        // Wait until the modal closes before reloading
        borrowPage.Disappearing += async (s, e) => await LoadInventory();
    }

    private async void OpenReturnPage(InventoryAvailability item)
    {
        var returnPage = new ReturnPage(item);
        await Application.Current.MainPage.Navigation.PushModalAsync(returnPage);

        // Wait until the modal closes before reloading
        returnPage.Disappearing += async (s, e) => await LoadInventory();
    }

    private async Task OpenAddInventoryPage()
    {
        var addPage = new AddInventoryPage();
        await Application.Current.MainPage.Navigation.PushModalAsync(addPage);

        // Wait until the modal closes before reloading
        addPage.Disappearing += async (s, e) => await LoadInventory();
    }

    public event PropertyChangedEventHandler PropertyChanged;
}
```
---

# Transaction Views

## 📁 Views

### AddInventoryPage

#### AddInventoryPage.xaml 

```xml
<Grid VerticalOptions="Center" HorizontalOptions="Center">
    <Frame Padding="20"
           BackgroundColor="White"
           CornerRadius="15"
           WidthRequest="340">

        <VerticalStackLayout Spacing="15">

            <Label Text="Add Inventory Item"
                   FontSize="20"
                   TextColor="Black"
                   FontAttributes="Bold"
                   HorizontalOptions="Center" />

            <Grid ColumnDefinitions="2*, Auto" ColumnSpacing="10">
                <VerticalStackLayout Grid.Column="0">
                    <Label Text="Inventory Name"
                           FontAttributes="Bold"
                           TextColor="Gray" />
                    
                    <Frame BorderColor="LightGray"
                       CornerRadius="6"
                       Padding="0">
                    <Entry Placeholder="Enter name"
                           TextColor="Black"
                           BackgroundColor="White" 
                           Text="{Binding InventoryName}" />
                    </Frame>
                </VerticalStackLayout>

                <VerticalStackLayout Grid.Column="1">
                    <Label Text="Quantity"
                           FontAttributes="Bold"
                           TextColor="Gray" />
                    <Frame BorderColor="LightGray"
                       CornerRadius="6"
                       Padding="0"
                       WidthRequest="80">
                        <Entry Placeholder="0"
                           Keyboard="Numeric"
                           WidthRequest="80"
                           TextColor="Black"
                           BackgroundColor="White" 
                           Text="{Binding MaxQuantity}" />
                    </Frame>
                </VerticalStackLayout>
            </Grid>

            <VerticalStackLayout>
                <Label Text="Media Type"
                       FontAttributes="Bold"
                       TextColor="Gray" 
                       Margin="0,0,0,2" />
                
                <Frame BorderColor="LightGray"
                   CornerRadius="6"
                   Padding="0"
                   HeightRequest="38">
                    
                    <Grid ColumnDefinitions="*, Auto"
                          VerticalOptions="Center"
                          HeightRequest="38">

                        <!-- Picker must fill the entire cell -->
                        <Picker x:Name="MediaPicker"
                            ItemsSource="{Binding MediaTypes}"
                            SelectedItem="{Binding SelectedMediaType}"
                            BackgroundColor="White"
                            TextColor="Black"
                            Margin="6,0"
                            FontSize="13"
                            HeightRequest="34"
                            HorizontalOptions="Fill"
                            VerticalOptions="Center" />

                        <Path Data="M 0 0 L 10 0 L 5 6 Z"
                              Fill="White"
                              WidthRequest="12"
                              HeightRequest="12"
                              VerticalOptions="Center"
                              Margin="0,0,10,0"
                              Grid.Column="1"
                              InputTransparent="True">
                            <!-- ⭐ Important -->
                            <Path.GestureRecognizers>
                                <TapGestureRecognizer Tapped="OnArrowTapped" />
                            </Path.GestureRecognizers>
                        </Path>

                    </Grid>
                </Frame>
            </VerticalStackLayout>


            <Button Text="Add Item"
                    BackgroundColor="#4CAF50"
                    TextColor="White"
                    CornerRadius="8"
                    Command="{Binding AddCommand}" />

            <Button Text="Cancel"
                    BackgroundColor="#B39DDB"
                    TextColor="Black"
                    CornerRadius="8"
                    Command="{Binding CloseCommand}" />

        </VerticalStackLayout>

    </Frame>
</Grid>
```
#### AddInventoryPage.xaml.cs

```csharp
public partial class AddInventoryPage : ContentPage
{
	public AddInventoryPage()
	{
		InitializeComponent();
        BindingContext = new AddInventoryViewModel();
    }

    private void OnArrowTapped(object sender, TappedEventArgs e)
    {
        MediaPicker.Focus(); // Opens the dropdown
    }
}
```
---

### BorrowPage

#### BorrowPage.xaml
```xml
<Grid VerticalOptions="Center" HorizontalOptions="Center">
    <Frame Padding="20"
           BackgroundColor="White"
           CornerRadius="15"
           WidthRequest="300">

        <VerticalStackLayout Spacing="15">

            <Label Text="Borrow Item for"
                   FontSize="20"
                   TextColor="Black"
                   FontAttributes="Bold"
                   HorizontalOptions="Center" />
            <Label Text="{Binding SelectedItem.inventory_name}"
                   FontSize="18"
                   HorizontalOptions="Center"
                   TextColor="Gray" />

            <Grid ColumnDefinitions="*, *" ColumnSpacing="10" VerticalOptions="Center">
                <!-- Remaining -->
                <VerticalStackLayout Grid.Column="0" HorizontalOptions="Center">
                    <Label Text="Remaining"
           TextColor="Gray"
           HorizontalOptions="Center"
           VerticalOptions="Center" />
                    <Frame BorderColor="LightGray"
           CornerRadius="6"
           Padding="0"
           HeightRequest="40"
           WidthRequest="100"
           BackgroundColor="White"
           HorizontalOptions="Center"
           VerticalOptions="Center">
                        <Label Text="{Binding SelectedItem.remaining_quantity}"
               FontAttributes="Bold"
               TextColor="Green"
               HorizontalOptions="Center"
               VerticalOptions="Center" />
                    </Frame>
                </VerticalStackLayout>

                <!-- Borrowing -->
                <VerticalStackLayout Grid.Column="1" HorizontalOptions="Center">
                    <Label Text="Borrowing"
           TextColor="Gray"
           HorizontalOptions="Center"
           VerticalOptions="Center" />
                    <Frame BorderColor="LightGray"
           CornerRadius="6"
           Padding="0"
           HeightRequest="40"
           WidthRequest="100"
           BackgroundColor="White"
           HorizontalOptions="Center"
           VerticalOptions="Center">
                        <Entry Placeholder="Enter quantity"
               Keyboard="Numeric"
               TextColor="Black"
               BackgroundColor="Transparent"
               HorizontalOptions="Center"
               VerticalOptions="Center"
               Text="{Binding Quantity}" />
                    </Frame>
                </VerticalStackLayout>
            </Grid>

            <!-- Borrower Selector -->
            <VerticalStackLayout Spacing="5">
                <!-- Search Bar -->
                <SearchBar Placeholder="Search borrower"
                   Text="{Binding SearchText}"
                   TextColor="Gray"/>

                            <!-- Filtered User List -->
                <CollectionView ItemsSource="{Binding FilteredUsers}"
                        SelectionMode="Single"
                        SelectedItem="{Binding SelectedUser}"
                        IsVisible="{Binding IsSearching}">
                                <CollectionView.ItemTemplate>
                                    <DataTemplate>
                                        <Frame Padding="10" Margin="0,5" BorderColor="LightGray">
                                            <Label Text="{Binding name}" />
                                        </Frame>
                                    </DataTemplate>
                                </CollectionView.ItemTemplate>
                            </CollectionView>

                <!-- Selected User Display -->
                            <Label Text="{Binding SelectedUser.name, StringFormat='Borrower: {0}'}"
               TextColor="Gray"
               HorizontalOptions="Center" />
            </VerticalStackLayout>

            <Button Text="Confirm Borrow"
                    BackgroundColor="#4CAF50"
                    TextColor="White"
                    Command="{Binding ConfirmBorrowCommand}" />
             

            <Button Text="Cancel"
                    Command="{Binding CloseCommand}" />

        </VerticalStackLayout>

    </Frame>
</Grid>
```
#### BorrowPage.xaml.cs
```csharp
public partial class BorrowPage : ContentPage
{
    public BorrowPage(InventoryAvailability item)
    {
        InitializeComponent();
        BindingContext = new BorrowPageViewModel(item);
    }
}
```
---
### ReturnPage

#### ReturnPage.xaml
```xml
 <Grid VerticalOptions="Center" HorizontalOptions="Center">
    <Frame Padding="20"
           BackgroundColor="White"
           CornerRadius="15"
           WidthRequest="320">

        <VerticalStackLayout Spacing="15">

            <Label Text="Return Item"
                   FontSize="20"
                   FontAttributes="Bold"
                   HorizontalOptions="Center" />

            <!-- Search Bar -->
            <SearchBar Placeholder="Search borrower"
                       Text="{Binding SearchText}"
                       TextColor="Gray"/>

            <!-- Filtered Borrowed Users -->
            <CollectionView ItemsSource="{Binding FilteredBorrowedUsers}"
                            SelectionMode="Single"
                            SelectedItem="{Binding SelectedBorrowedUser}"
                            IsVisible="{Binding IsSearching}">
                <CollectionView.ItemTemplate>
                    <DataTemplate>
                        <Frame Padding="10" Margin="0,5" BorderColor="LightGray">
                            <VerticalStackLayout>
                                <Label Text="{Binding user_name}" FontAttributes="Bold" />
                                <Label Text="{Binding borrow_id}" FontSize="12" TextColor="Gray" />
                                <Label Text="{Binding quantity, StringFormat='Qty: {0}'}" FontSize="12" TextColor="Gray" />
                            </VerticalStackLayout>
                        </Frame>
                    </DataTemplate>
                </CollectionView.ItemTemplate>
            </CollectionView>

            <!-- Selected Borrow Info -->
            <Label Text="{Binding SelectedBorrowedUser.user_name, StringFormat='Borrower: {0}'}"
                   TextColor="Gray"
                   HorizontalOptions="Center" />

            <Label Text="{Binding SelectedBorrowedUser.quantity, StringFormat='Borrowed Quantity: {0}'}"
                   FontAttributes="Bold"
                   HorizontalOptions="Center"
                   TextColor="Black" />

            <Button Text="Confirm Return"
                    BackgroundColor="#2196F3"
                    TextColor="White"
                    Command="{Binding ConfirmReturnCommand}" />

            <Button Text="Cancel"
                    Command="{Binding CloseCommand}" />

        </VerticalStackLayout>
    </Frame>
</Grid>
```
#### UsersPage.xaml.cs
```csharp
public partial class ReturnPage : ContentPage
{
	public ReturnPage(InventoryAvailability item)
	{
		InitializeComponent();
        BindingContext = new ReturnPageViewModel(item);
    }
}
```
---
### TransactionPage

#### TransactionPage.xaml
```xml
 <ContentPage.ToolbarItems>
     <ToolbarItem Text="Add"
                  Order="Primary"
                  Priority="0"
                  Command="{Binding AddInventoryCommand}" />
 </ContentPage.ToolbarItems>
 
 <CollectionView ItemsSource="{Binding Inventory}">
     <CollectionView.Header>
         <Grid ColumnDefinitions="4*, *, *, *, *"
           Padding="10"
           BackgroundColor="Black">
             <Label Grid.Column="0" Text="Inventory" FontAttributes="Bold" TextColor="White"/>
             <Label Grid.Column="1" Text="Max" FontAttributes="Bold" TextColor="White"/>
             <Label Grid.Column="2" Text="Remaining" FontAttributes="Bold" TextColor="White"/>
             <Label Grid.Column="3" Text="Borrow" FontAttributes="Bold" TextColor="White"/>
             <Label Grid.Column="4" Text="Return" FontAttributes="Bold" TextColor="White"/>
         </Grid>
     </CollectionView.Header>

     <CollectionView.ItemTemplate>
         <DataTemplate>
             <Grid ColumnDefinitions="4*, *, *, *, *" Padding="10" BackgroundColor="#1E1E1E">

                 <Label Grid.Column="0" Text="{Binding inventory_name}" TextColor="White"/>
                 <Label Grid.Column="1" Text="{Binding max_quantity}" TextColor="White"/>
                 <Label Grid.Column="2" Text="{Binding remaining_quantity}" TextColor="White"/>

                 <Button Grid.Column="3"
                         Text="Borrow"
                         BackgroundColor="#F44336"
                         TextColor="White"
                         Command="{Binding Source={RelativeSource AncestorType={x:Type ContentPage}}, Path=BindingContext.OpenBorrowCommand}"
                         CommandParameter="{Binding}" />

                 <Button Grid.Column="4"
                         Text="Return"
                         BackgroundColor="#4CAF50"
                         TextColor="White"
                         Command="{Binding Source={RelativeSource AncestorType={x:Type ContentPage}}, Path=BindingContext.OpenReturnCommand}"
                         CommandParameter="{Binding}" />

             </Grid>
         </DataTemplate>
     </CollectionView.ItemTemplate>
 </CollectionView>
```
#### TransactionPage.xaml.cs
```csharp
public partial class TransactionPage : ContentPage
{
	public TransactionPage()
    {
		InitializeComponent();
        BindingContext = new TransactionViewModel();
    }
}
```
---

# Edit Navigation

Add this to your AppShell.xml

#### AppShell.xaml
```xml
        <!-- TRANSACTIONS TAB -->
        <ShellContent
            Title="Transactions"
            ContentTemplate="{DataTemplate local:TransactionPage}"
            Route="TransactionPage" />
```

---

## 🔗 Navigation

[← Previous (Users)](page2.md) | [Next → (Page 3)](page4.md)
