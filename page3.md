# 📄 Creating Transactions

In this section, we will create the **Transaction Page** that represent the data structures used throughout the application.  
Models define the shape of your data and are used by services, viewmodels, and views.

## 📑 Table of Contents
- [1. User Model](#User-Models)
- [2. User Services](#User-CRUD-Services)
- [3. View Models](#User-ViewModels)
- [4. View Models](#User-Views)
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

### InventoryAvailability.cs
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

# User ViewModels

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
---

# Transaction Views

## 📁 Views

### InsertUserPage

#### InsertUserPage.xaml 

```xml
<Grid VerticalOptions="Center" HorizontalOptions="Center">
    <Frame Padding="20"
           BackgroundColor="White"
           CornerRadius="15"
           WidthRequest="340">

        <VerticalStackLayout Spacing="15">

            <Label Text="Add User"
                   FontSize="20"
                   TextColor="Black"
                   FontAttributes="Bold"
                   HorizontalOptions="Center" />

            <!-- Name -->
            <VerticalStackLayout>
                <Label Text="Name" FontAttributes="Bold" TextColor="Gray" />
                <Frame BorderColor="LightGray" CornerRadius="6" Padding="0">
                    <Entry Placeholder="Enter name"
                           Text="{Binding Name}"
                           BackgroundColor="White" 
                           TextColor="Black" />
                </Frame>
            </VerticalStackLayout>

            <!-- Email -->
            <VerticalStackLayout>
                <Label Text="Email" FontAttributes="Bold" TextColor="Gray" />
                <Frame BorderColor="LightGray" CornerRadius="6" Padding="0">
                    <Entry Placeholder="Enter email"
                           Keyboard="Email"
                           Text="{Binding Email}"
                           BackgroundColor="White" 
                           TextColor="Black" />
                </Frame>
            </VerticalStackLayout>

            <!-- Age -->
            <VerticalStackLayout>
                <Label Text="Age" FontAttributes="Bold" TextColor="Gray" />
                <Frame BorderColor="LightGray" CornerRadius="6" Padding="0">
                    <Entry Placeholder="Enter age"
                           Keyboard="Numeric"
                           Text="{Binding Age}"
                           BackgroundColor="White" 
                           TextColor="Black" />
                </Frame>
            </VerticalStackLayout>

            <Button Text="Add User"
                    BackgroundColor="#4CAF50"
                    TextColor="White"
                    CornerRadius="8"
                    Command="{Binding InsertUserCommand}" />

            <Button Text="Cancel"
                    BackgroundColor="#B39DDB"
                    TextColor="Black"
                    CornerRadius="8"
                    Command="{Binding CloseCommand}" />

        </VerticalStackLayout>

    </Frame>
</Grid>
```
#### InsertUserPage.xaml.cs

```csharp
public partial class InsertUserPage : ContentPage
{
	public InsertUserPage()
	{
        InitializeComponent();
        BindingContext = new InsertUserViewModel();
    }

}
```
---

### ModifyUserPage

#### ModifyUserPage.xaml
```xml
<Grid VerticalOptions="Center" HorizontalOptions="Center">
    <Frame Padding="20" BackgroundColor="White" CornerRadius="15" WidthRequest="340">

        <VerticalStackLayout Spacing="15">

            <Label Text="Modify User"
                   FontSize="20"
                   FontAttributes="Bold"
                   HorizontalOptions="Center" />

            <VerticalStackLayout>
                <Label Text="Name" FontAttributes="Bold" TextColor="Gray" />
                <Frame BorderColor="LightGray" CornerRadius="6" Padding="0">
                    <Entry Text="{Binding Name}" BackgroundColor="White" TextColor="Black" />
                </Frame>
            </VerticalStackLayout>

            <VerticalStackLayout>
                <Label Text="Email" FontAttributes="Bold" TextColor="Gray" />
                <Frame BorderColor="LightGray" CornerRadius="6" Padding="0">
                    <Entry Text="{Binding Email}" BackgroundColor="White" TextColor="Black" />
                </Frame>
            </VerticalStackLayout>

            <VerticalStackLayout>
                <Label Text="Age" FontAttributes="Bold" TextColor="Gray" />
                <Frame BorderColor="LightGray" CornerRadius="6" Padding="0">
                    <Entry Text="{Binding Age}" Keyboard="Numeric" BackgroundColor="White" TextColor="Black" />
                </Frame>
            </VerticalStackLayout>

            <Button Text="Save Changes"
                    BackgroundColor="#4CAF50"
                    TextColor="White"
                    CornerRadius="8"
                    Command="{Binding UpdateUserCommand}" />

            <Button Text="Cancel"
                    BackgroundColor="#B39DDB"
                    TextColor="Black"
                    CornerRadius="8"
                    Command="{Binding CloseCommand}" />

        </VerticalStackLayout>

    </Frame>
</Grid>
```
#### ModifyUserPage.xaml.cs
```csharp
public partial class ModifyUserPage : ContentPage
{
	public ModifyUserPage()
	{
		InitializeComponent();
    }

    public ModifyUserPage(user user)
    {
        InitializeComponent();
        BindingContext = new ModifyUserViewModel(user);
    }
}
```
---
### UsersPage

#### UsersPage.xaml
```xml
 <ContentPage.ToolbarItems>
     <ToolbarItem Text="Add"
                  Order="Primary"
                  Priority="0"
                  Command="{Binding OpenAddUserCommand}" />
 </ContentPage.ToolbarItems>
 
 <CollectionView x:Name="UsersList"
             ItemsSource="{Binding Users}">

     <!-- HEADER ROW -->
     <CollectionView.Header>
         <Grid ColumnDefinitions="2*, 2*, *, *, *" Padding="8" BackgroundColor="Black" >
             <Label Grid.Column="0" Text="Name" FontAttributes="Bold" />
             <Label Grid.Column="1" Text="Email" FontAttributes="Bold" />
             <Label Grid.Column="2" Text="Age" FontAttributes="Bold" />
             <Label Grid.Column="3" Text="Modify" FontAttributes="Bold" />
             <Label Grid.Column="4" Text="Delete" FontAttributes="Bold" />
         </Grid>
     </CollectionView.Header>

     <!-- TABLE BODY -->
     <CollectionView.ItemTemplate>
         <DataTemplate>
             <Grid ColumnDefinitions="2*, 2*, *, *, *" Padding="8">
                 <Label Grid.Column="0" Text="{Binding name}" />
                 <Label Grid.Column="1" Text="{Binding email}" />
                 <Label Grid.Column="2" Text="{Binding age}" />
                 <!-- Modify -->
                 <Button Grid.Column="3"
                         Text="Edit"
                         Padding="5"
                         Command="{Binding Source={RelativeSource AncestorType={x:Type ContentPage}}, Path=BindingContext.OpenModifyUserCommand}"
                         CommandParameter="{Binding}" />

                 <!-- Delete -->
                 <Button Grid.Column="4"
                         Text="Delete"
                         Padding="5"
                         BackgroundColor="#FFCDD2"
                         TextColor="Black"
                         Command="{Binding Source={RelativeSource AncestorType={x:Type ContentPage}}, Path=BindingContext.DeleteUserCommand}"
                         CommandParameter="{Binding}" />
                
             </Grid>
         </DataTemplate>
     </CollectionView.ItemTemplate>

 </CollectionView>
```
#### UsersPage.xaml.cs
```csharp
public partial class UsersPage : ContentPage
{
    private readonly UsersPageViewModel _viewModel;

    public UsersPage()
    {
        InitializeComponent();
        _viewModel = new UsersPageViewModel();
        BindingContext = _viewModel;

        BindingContextChanged += async (s, e) =>
        {
            await _viewModel.LoadUsers();
        };
    }

    protected override async void OnAppearing()
    {
        base.OnAppearing();
        await _viewModel.LoadUsers();
    }
}
```

---

# Edit Navigation

#### AppShell.xaml
```xml
         <!-- HOME TAB -->
        <ShellContent
        Title="Home"
        ContentTemplate="{DataTemplate local:MainPage}" 
        Route="MainPage"/>

        <!-- USER TAB -->
        <ShellContent
        Title="Users"
        ContentTemplate="{DataTemplate local:UsersPage}" 
        Route="UsersPage"/>
```

---

## 🔗 Navigation

[← Previous (Page 1)](README.md) | [Next → (Page 3)](page3.md)
