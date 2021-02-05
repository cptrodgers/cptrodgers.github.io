---
published: true
---
This is a note for someone who is finding a way that writes main logic by Rust and using in C# (or other language). Thanks for great post by Test Double ([https://blog.testdouble.com/posts/2018-01-02-unity-rust-ffi-getting-started](https://blog.testdouble.com/posts/2018-01-02-unity-rust-ffi-getting-started))

# Why we need communicate with other language?

It's easy to become an over-engineer decision when you want to write some logic in a programming language and execute it in another programming language. You need to think carefully about this decision. It's useful when:

- You want to share logic between 2 services (Game Server in Rust - Game Client in C#). For example, Prediction when moving.
- You want to re-use your legacy library with poor support from the host language. For example, networking.
- You want to utilize the unique power of some language that the host language cannot provide. For example, Memory Safety by Rust.

# What is FFI?

FFI (Foreign Function Interface) is the way that extern the function interface that another language can call. For example, you want to write a function in Rust and call this function in C#. You need to declare a function with FFI, compile to a dynamic library, import it to C#.

# Rules to make your idea works as well

- Declare it as FFI function (for sure).

    ```rust
    pub extern "C" fn example() {
    	prinln!("Hello FFI");
    }
    ```

- Request your compiler doesn't change the function name. If the name of function changed after compiled, host language cannot call it. In rust, you need use `#[no_mangle]`

    ```rust
    #[no_mangle]
    pub extern "C" fn example() {
    	prinln!("Hello FFI");
    }
    ```

    ```csharp
    public class FFI {
    	/// Declare dylib/dll that we want to import
    	[DllImport("libexample")]
    	private static extern void example(); /// Declare function that we want to call
    }
    ```

- Use Pointer to make complicated struct (Vector, HasMap, and so on):

    ```rust
    #[derive(Debug, Clone)]
    pub struct Players {
    	player_ids: Vec<i32>,
    }

    #[no_mangle]
    pub extern "C" fn init_players(ptr: *mut *const Players) {
    	unsafe { *ptr = transmute(Box::new(vec![])) }
    }

    #[no_mangle]
    pub extern "C" fn add_player(ptr: *mut Players, player_id: i32) {
    	if ptr.is_null() {
    		return;
    	}

    	unsafe { (*ptr).player_ids.push(player_id) }
    }
    ```

    ```csharp
    public class FFI {
    	/// Declare Pointer that address to Player struct in Rust code
    	private static IntPtr _players;
    	
    	[DllImport("libexample")]
    	private static extern void init_players(out IntPtr players);
    	public static void InitPlayers() {
    		init_players(out _players);
    	}

    	[DllImport("libexample")]
    	private static extern void add_player(IntPtr players, Int32 player_id);
    	public static void addPlayer(Int32 player_id) {
    		add_player(_players, player_id);
    	}
    }
    ```

- Return and receive value: Use same layout of memory by using middle Language such as C layout.

    ```rust
    #[derive(Debug, Clone, Copy)]
    #[repr(C)] // Mark Player should be stored by C layout
    pub struct Player {
    	pub player_id: i32,
    	pub score: i32,
    }

    impl Player {
    	pub fn from_ptr(ptr: *mut Player) -> Player {
    		unsafe { *ptr }
    	}
    }

    #[derive(Debug, Clone)]
    pub struct Players {
    	items: HashMap<i32, Player>, // (player id, player)
    }

    impl<'a> Players {
    	pub fn from_ptr(ptr: *mut Players) -> &'a mut Players {
    		unsafe { &mut *ptr }
    	}

    	pub fn to_ptr(self) -> *mut Players {
    		unsafe { transmute(Box::new(self)) }
    	}

    	pub fn new() -> Self {
    		Self { items: HashMap::new() }
    	}
    }

    #[no_mangle]
    pub extern "C" fn init_players(ptr: *mut *const Players) {
    	unsafe { *ptr = Player::new().to_ptr() }
    }

    #[no_mangle]
    pub extern "C" fn add_player(ptr: *mut Players, player_ptr: *mut Player) {
    	if ptr.is_null() || player.is_null() {
    		return;
    	}
    	let player = Player::from_ptr(player_ptr);
    	Players::from_ptr(ptr).insert(player.player_id, player);
    }

    #[no_mangle]
    pub extern "C" fn get_player(ptr: *mut Players, player_id: i32) -> *mut Player {
    	if ptr.is_null() {
    		return;
    	}

    	let default_player = Player { player_id: -1, score: 0 };
    	Players::from_ptr(ptr)
    		.items
    		.get(&player_id)
    		.cloned()
    		.unwrap_or(default_player)
    }
    ```

    ```csharp
    public class FFI {
    	/// Declare Pointer that address to Player struct in Rust code
    	private static IntPtr _players;

    	/// Make sure Player Class store in C layout.
    	[StructLayout(LayoutKind.Sequential)]
    	public class Player {
    		public Int32 player_id;
    		public Int32 score;
    	}
    	
    	[DllImport("libexample")]
    	private static extern void init_players(out IntPtr players);
    	public static void InitPlayers() {
    		init_players(out _players);
    	}

    	[DllImport("libexample")]
    	private static extern void add_player(IntPtr players, Player player);
    	public static void AddPlayer(Player player) {
    		add_player(_players, player);
    	}

    	[DllImport("libexample")]
    	private static extern Player get_player(IntPtr players, Int32 player_id);
    	public static Player GetPlayer(Int32 player_id) {
    		return get_player(_players, player_id);
    	}
    }
    ```

# Github example

Update soon...
