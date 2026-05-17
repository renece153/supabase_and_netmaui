# 📄 Creating your Users

In this section, we will create the **User Page** that represent the data structures used throughout the application.  
Models define the shape of your data and are used by services, viewmodels, and views.

## 📑 Table of Contents
- [1. User Model](#Models)
- [2. User Services](#Services)
- [3. Organizing your files](#3-organizing-your-files)


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

# 📁 Models

## 📦 users.cs

```csharp
public class user
{
    public long id { get; set; }
    public string name { get; set; }
    public string email { get; set; }
    public int age { get; set; }
}
```

# 📁 Services

## UserCRUDServices.cs

```csharp
public class UserCRUDServices
{
    private readonly HttpClient _client;

    public UserCRUDServices()
    {
        _client = new HttpClient();
        _client.DefaultRequestHeaders.Add("apikey", Constants.SupabaseKey);
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", Constants.SupabaseKey);
    }

    public async Task<string> GetUsersJsonAsync()
    {
        var url = $"{Constants.SupabaseUrl}/rest/v1/users?select=*";
        var response = await _client.GetAsync(url);
        return await response.Content.ReadAsStringAsync();
    }

    public async Task InsertUserAsync(object user)
    {
        var jsonBody = JsonSerializer.Serialize(user);
        var content = new StringContent(jsonBody, Encoding.UTF8, "application/json");

        var url = $"{Constants.SupabaseUrl}/rest/v1/users";
        await _client.PostAsync(url, content);
    }

    public async Task UpdateUserAsync(long id, object body)
    {
        var jsonBody = JsonSerializer.Serialize(body);
        var content = new StringContent(jsonBody, Encoding.UTF8, "application/json");

        var url = $"{Constants.SupabaseUrl}/rest/v1/users?id=eq.{id}";
        await _client.PatchAsync(url, content);
    }

    public async Task DeleteUserAsync(long id)
    {
        var url = $"{Constants.SupabaseUrl}/rest/v1/users?id=eq.{id}";
        await _client.DeleteAsync(url);
    }

}
```
---

# 📁 ViewModels

## InsertUserViewModel.cs

```csharp
public class InsertUserViewModel : INotifyPropertyChanged
{
    private readonly UserCRUDServices _service;

    public string Name { get; set; }
    public string Email { get; set; }
    public string Age { get; set; }

    public Command InsertUserCommand { get; }
    public Command CloseCommand { get; }

    private bool _isInserting;

    public InsertUserViewModel()
    {
        _service = new UserCRUDServices();

        InsertUserCommand = new Command(async () => await InsertUser());
        CloseCommand = new Command(async () => await Close());
    }

    private async Task InsertUser()
    {
        if (_isInserting) return;
        _isInserting = true;

        try
        {
            var user = new
            {
                name = Name,
                email = Email,
                age = int.Parse(Age)
            };

            await _service.InsertUserAsync(user);

            await Application.Current.MainPage.DisplayAlert("Success", "User added!", "OK");
            await Close();
        }
        finally
        {
            _isInserting = false;
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

## ModifyUserViewModel.cs

```csharp
public class ModifyUserViewModel : INotifyPropertyChanged
{
    private readonly UserCRUDServices _service;
    private readonly long _userId;

    public string Name { get; set; }
    public string Email { get; set; }
    public string Age { get; set; }

    public Command UpdateUserCommand { get; }
    public Command CloseCommand { get; }

    public ModifyUserViewModel(user user)
    {
        _service = new UserCRUDServices();

        _userId = user.id;
        Name = user.name;
        Email = user.email;
        Age = user.age.ToString();

        UpdateUserCommand = new Command(async () => await UpdateUser());
        CloseCommand = new Command(async () => await Close());
    }

    private async Task UpdateUser()
    {
        var body = new
        {
            name = Name,
            email = Email,
            age = int.Parse(Age)
        };

        await _service.UpdateUserAsync(_userId, body);

        await Application.Current.MainPage.DisplayAlert("Success", "User updated!", "OK");
        await Close();
    }

    private async Task Close()
    {
        await Application.Current.MainPage.Navigation.PopModalAsync();
    }

    public event PropertyChangedEventHandler PropertyChanged;
}
```

## UsersPageViewModel.cs

```csharp
public class UsersPageViewModel : INotifyPropertyChanged
{
    private readonly UserCRUDServices _service;

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

    public Command OpenAddUserCommand { get; }
    public Command OpenModifyUserCommand { get; }
    public Command DeleteUserCommand { get; }

    public UsersPageViewModel()
    {
        _service = new UserCRUDServices();

        OpenAddUserCommand = new Command(async () => await OpenAddUser());
        OpenModifyUserCommand = new Command<user>(async (u) => await OpenModifyUser(u));
        DeleteUserCommand = new Command<user>(async (u) => await DeleteUser(u));

        _ = LoadUsers();
    }

    // -----------------------------
    // Load Users
    // -----------------------------
    public async Task LoadUsers()
    {
        var json = await _service.GetUsersJsonAsync();
        var users = System.Text.Json.JsonSerializer.Deserialize<List<user>>(json);

        Users = new ObservableCollection<user>(users);
    }

    // -----------------------------
    // Add User Modal
    // -----------------------------
    private async Task OpenAddUser()
    {
        var page = new InsertUserPage();
        await Application.Current.MainPage.Navigation.PushModalAsync(page);

        // Refresh when modal closes
        page.Disappearing += async (s, e) => await LoadUsers();
    }

    // -----------------------------
    // Modify User Modal
    // -----------------------------
    private async Task OpenModifyUser(user user)
    {
        var page = new ModifyUserPage(user);
        await Application.Current.MainPage.Navigation.PushModalAsync(page);

        // Refresh when modal closes
        page.Disappearing += async (s, e) => await LoadUsers();
    }

    // -----------------------------
    // Delete User
    // -----------------------------
    private async Task DeleteUser(user user)
    {
        bool confirm = await Application.Current.MainPage.DisplayAlert(
            "Delete User",
            $"Are you sure you want to delete {user.name}?",
            "Yes", "No");

        if (!confirm) return;

        await _service.DeleteUserAsync(user.id);
        await LoadUsers();
    }

    // -----------------------------
    // PropertyChanged
    // -----------------------------
    public event PropertyChangedEventHandler PropertyChanged;
    private void OnPropertyChanged(string name)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
    }
}
```
---

# 📁 Views

## InsertUserPage

### InsertUserPage.xaml 

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
### InsertUserPage.xaml.cs

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

## ModifyUserPage

### ModifyUserPage.xaml
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
### ModifyUserPage.xaml.cs
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
## UsersPage

### UsersPage.xaml
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
### UsersPage.xaml.cs
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


---

## 🔗 Navigation

[← Previous (Page 1)](README.md) | [Next → (Page 3)](page3.md)
