using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;

namespace LibraryApp
{
    public abstract class User
    {
        public string Name { get; protected set; }
        public abstract void ShowMenu(Library library);
    }

    public class Librarian : User
    {
        private string password;

        public Librarian(string name, string password)
        {
            Name = name;
            this.password = password;
        }

        public string Password { get => password; }

        public bool VerifyPassword(string password)
        {
            return this.password == password;
        }

        public override void ShowMenu(Library library)
        {
            while (true)
            {
                Console.WriteLine("\nМеню:");
                Console.WriteLine("1. Добавить книгу");
                Console.WriteLine("2. Удалить книгу");
                Console.WriteLine("3. Добавить пользователя"); 
                Console.WriteLine("4. Посмотреть всех пользователей");
                Console.WriteLine("5. Посмотреть все книги");
                Console.WriteLine("6. ВЫход");

                string choice = Console.ReadLine();

                switch (choice)
                {
                    case "1": library.AddBook(); break;
                    case "2": library.RemoveBook(); break;
                    case "3": library.AddUser(); break; 
                    case "4": library.ViewAllUsers(); break;
                    case "5": library.ViewAllBooks(); break;
                    case "6": return;
                    default: Console.WriteLine("Некорректный ввод."); break;
                }
            }
        }
    }

    public class LibraryUser : User
    {
        public List<string> BorrowedBooks { get; set; } = new List<string>();

        public LibraryUser(string name)
        {
            Name = name;
        }

        public override void ShowMenu(Library library)
        {
            while (true)
            {
                Console.WriteLine("\nМеню:");
                Console.WriteLine("1. Доступные книги");
                Console.WriteLine("2. Взять книгу");
                Console.WriteLine("3. Вернуть книгу");
                Console.WriteLine("4. Посмотреть одолженные книги");
                Console.WriteLine("5. Вход");

                string choice = Console.ReadLine();

                switch (choice)
                {
                    case "1": library.ViewAvailableBooks(); break;
                    case "2": library.BorrowBook(this); break;
                    case "3": library.ReturnBook(this); break;
                    case "4": ViewBorrowedBooks(); break;
                    case "5": return;
                    default: Console.WriteLine("Неккоректный ввод."); break;
                }
            }
        }

        void ViewBorrowedBooks()
        {
            Console.WriteLine("\nОдлженные книги:");
            if (BorrowedBooks.Count == 0) Console.WriteLine("Нет одолженных книг");
            else BorrowedBooks.ForEach(book => Console.WriteLine($"- {book}"));
        }
    }

    public class Book
    {
        public string Title { get; set; }
        public string Author { get; set; }
        public bool IsAvailable { get; set; } = true;

        public override string ToString() => $"{Title} by {Author} ({(IsAvailable ? "Доступно" : "Одолженно")})";
    }

    public class Library
    {
        private List<Book> books = new();
        private List<LibraryUser> users = new();
        private List<Librarian> librarians = new();
        private const string BooksFile = "books.txt";
        private const string UsersFile = "users.txt";
        private const string LibrariansFile = "librarians.txt";

        public Library()
        {
            librarians.Add(new Librarian("Admin", "password123"));
            LoadData();
        }

        void LoadData()
        {
            LoadBooks();
            LoadUsers();
            LoadLibrarians();
        }
        void LoadLibrarians()
        {
            if (!File.Exists(LibrariansFile)) return;
            File.ReadAllLines(LibrariansFile)
                .Select(line => line.Split('|'))
                .Where(parts => parts.Length == 2)
                .ToList().ForEach(parts =>
                {
                    if (parts[0] == "Admin" && parts[1] == "password123")
                        return;
                    librarians.Add(new Librarian(parts[0], parts[1]));
                });
        }


        void LoadBooks()
        {
            if (!File.Exists(BooksFile)) return;
            File.ReadAllLines(BooksFile)
                .Select(line => line.Split('|'))
                .Where(parts => parts.Length == 3)
                .ToList().ForEach(parts =>
                {
                    if (bool.TryParse(parts[2], out bool isAvailable))
                    {
                        books.Add(new Book { Title = parts[0], Author = parts[1], IsAvailable = isAvailable });
                    }
                    else
                    {
                        Console.WriteLine($"Неверное значние: {parts[2]}.");
                        books.Add(new Book { Title = parts[0], Author = parts[1], IsAvailable = true });
                    }
                });
        }

        void LoadUsers()
        {
            if (!File.Exists(UsersFile)) return;
            File.ReadAllLines(UsersFile)
                .Select(line => line.Split('|'))
                .Where(parts => parts.Length >= 1)
                .ToList().ForEach(parts =>
                {
                    string name = parts[0];
                    if (string.IsNullOrWhiteSpace(name))
                    {
                        Console.WriteLine("Такого пользователя нет");
                        return;
                    }
                    var user = new LibraryUser(name);
                    users.Add(user);
                    if (parts.Length > 1)
                    {
                        user.BorrowedBooks.AddRange(parts[1].Split(','));
                    }
                });
        }

        public void SaveData()
        {
            File.WriteAllLines(BooksFile, books.Select(b => $"{b.Title}|{b.Author}|{b.IsAvailable}"));
            File.WriteAllLines(UsersFile, users.Select(u => $"{u.Name}|{string.Join(",", u.BorrowedBooks)}"));
            File.WriteAllLines(LibrariansFile, librarians.Select(l => $"{l.Name}|{l.Password}"));
        }

        public void AddBook()
        {
            Console.Write("Введите название: ");
            string title = Console.ReadLine();
            Console.Write("Введите автора: ");
            string author = Console.ReadLine();
            if (string.IsNullOrWhiteSpace(title) || string.IsNullOrWhiteSpace(author))
            {
                Console.WriteLine("Вы не ввели название/автора");
                return;
            }
            books.Add(new Book { Title = title, Author = author });
            Console.WriteLine("Книга добавлена.");
        }

        public void RemoveBook()
        {
            Console.Write("Введите название: ");
            string title = Console.ReadLine();
            Book bookToRemove = books.FirstOrDefault(b => b.Title.Equals(title, StringComparison.OrdinalIgnoreCase));
            if (bookToRemove != null) { books.Remove(bookToRemove); Console.WriteLine("Книга удалена."); }
            else Console.WriteLine("Такой книги нет.");
        }

        public void AddUser()
        {
            Console.Write("Введите имя пользователя: ");
            string name = Console.ReadLine();
            if (string.IsNullOrWhiteSpace(name))
            {
                Console.WriteLine("Вы не ввели имя пользователя");
                return;
            }
            if (GetUserByName(name) != null)
            {
                Console.WriteLine("ПОльзователь с таким именем уже есть.");
                return;
            }
            users.Add(new LibraryUser(name));
            Console.WriteLine("Пользователь добавлен.");
        }

        public void ViewAllUsers() => users.ForEach(u => Console.WriteLine($"- {u.Name}"));

        public void ViewAllBooks() => books.ForEach(book => Console.WriteLine($"- {book}"));

        public void ViewAvailableBooks() => books.Where(b => b.IsAvailable).ToList().ForEach(book => Console.WriteLine($"- {book.Title} {book.Author}"));

        public void BorrowBook(LibraryUser user)
        {
            Console.Write("Введите название: ");
            string title = Console.ReadLine();
            Book bookToBorrow = books.FirstOrDefault(b => b.Title.Equals(title, StringComparison.OrdinalIgnoreCase) && b.IsAvailable);

            if (bookToBorrow != null)
            {
                bookToBorrow.IsAvailable = false;
                user.BorrowedBooks.Add(bookToBorrow.Title);
                Console.WriteLine("Книга одолжена.");
            }
            else Console.WriteLine("Такой книги нет.");
        }

        public void ReturnBook(LibraryUser user)
        {
            Console.Write("Введиет название: ");
            string title = Console.ReadLine();
            Book bookToReturn = books.FirstOrDefault(b => b.Title.Equals(title, StringComparison.OrdinalIgnoreCase) && !b.IsAvailable);

            if (bookToReturn != null && user.BorrowedBooks.Contains(bookToReturn.Title))
            {
                bookToReturn.IsAvailable = true;
                user.BorrowedBooks.Remove(bookToReturn.Title);
                Console.WriteLine("Книга сдана.");
            }
            else Console.WriteLine("Такой книги нет.");
        }
        public List<LibraryUser> GetUsers() => users;

        public List<Librarian> GetLibrarians() => librarians;

        public LibraryUser GetUserByName(string userName) => users.FirstOrDefault(u => u.Name.Equals(userName, StringComparison.OrdinalIgnoreCase));
        public Librarian GetLibrarianByName(string librarianName) => librarians.FirstOrDefault(u => u.Name.Equals(librarianName, StringComparison.OrdinalIgnoreCase));

        public void RegisterLibrarian(string name, string password) => librarians.Add(new Librarian(name, password));
    }

    class Program
    {
        static User Authenticate(Library library)
        {
            while (true)
            {
                Console.WriteLine("\nВойти как:");
                Console.WriteLine("1. Библиотекарь");
                Console.WriteLine("2. Пользователь");
                string choice = Console.ReadLine();

                switch (choice)
                {
                    case "1":
                        Console.Write("Логин: ");
                        string libName = Console.ReadLine();
                        Console.Write("Пароль: ");
                        string libPass = Console.ReadLine();
                        Librarian librarian = library.GetLibrarianByName(libName);

                        if (librarian != null && librarian.VerifyPassword(libPass))
                        {
                            return librarian;
                        }
                        else
                        {
                            Console.WriteLine("Данные не верны");
                            break;
                        }

                    case "2":
                        Console.Write("Имя пользователя: ");
                        string userName = Console.ReadLine();
                        LibraryUser user = library.GetUserByName(userName);
                        if (user == null)
                        {
                            Console.WriteLine("Пользователь не найден.");
                        }
                        return user;
                    default:
                        Console.WriteLine("Некорректный ввод.");
                        break;
                }
            }
        }

        static void Main(string[] args)
        {
            Library library = new Library();
            User currentUser;

            while (true)
            {
                Console.WriteLine("\nДобро пожаловать в библиотеку!");
                currentUser = Authenticate(library);
                if (currentUser != null)
                    currentUser.ShowMenu(library);
                else
                    Console.WriteLine("Авторизация не пройдена.");
                library.SaveData();
            }
        }
    }
}
