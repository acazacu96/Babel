
1. [Type Erasure](#Type%20Erasure)
	1. [Type Erasure with virtual functions](#Type%20Erasure%20with%20virtual%20functions)
	2. [Type erasure with free template functions and SSO](#Type%20erasure%20with%20free%20template%20functions%20and%20SSO)
	3. [GCC implementation of std::function](#GCC%20implementation%20of%20std::function)
	4. [SSO implementation of std::function](#SSO%20implementation%20of%20std::function)

## Type Erasure

### Type Erasure with virtual functions

- Abstract away the real type through inheritance and virtual functions
- example of a non-real world implementation of std::function that uses type erasure to store different callable types (e.g.: lamdas, function-pointer member-function-pointer)

``` c++
namespace acpp {

template <typename... Args>
class function;

template <typename R, typename... Args>
class function<R(Args...)> {
private:
	// Common interface for different kind of callables 
	class CallableConcept {
	public:
		virtual ~CallableConcept() = default;
		virtual R invoke(Args... args) = 0;
		virtual CallableConcept* clone() = 0;
	};
	// Actual class for wrapping the callables.
	// CallableModel<lambda_type_1>  |
	// CallableModel<lambda_type_2>  | is-a
	// CallableModel<R(*)(Args)>     | _ _ _ > CallableConcept* 
	// CallableModel<R(C::*)(Args)>  |
	template <typename F>
	class CallableModel : public CallableConcept {
	public:
		CallableModel(F f) : f_{std::move(f)} {}
		virtual R invoke(Args... args) override { 
			return std::invoke(f_, std::forward<Args>(args)...); 
		}
		virtual CallableConcept* clone() { return new CallableModel(f_); }
	private:
		F f_;
	};

	template <typename F>
	using CallableStorage = CallableModel<F>;

public:
	function() = default;
	function(const function& oth) {
		if (oth.callable_) { callable_.reset(oth.callable_->clone()); }
	}
	function& operator=(const function& oth) {
		if (this != &oth) { 
			callable_.reset(oth.callable_ ? oth.callable_->clone() : nullptr); 
		}
		return *this;
	}
	function(function&& oth) = default;
	function& operator=(function&& oth) = default;

	template <typename Callable>
	function(Callable callable):
		callable_{new CallableStorage<Callable>(std::move(callable))} {}
	
	template <typename Callable>
	function& operator=(Callable callable) { 
		callable_.reset(new CallableStorage<Callable>(std::move(callable))); 
		return *this;
	}

	R operator()(Args... args) {
		if (!callable_) { throw std::runtime_error("bad acpp::function call"); }
		return callable_->invoke(std::forward<Args>(args)...);
	}

private:
	std::unique_ptr<CallableConcept> callable_{nullptr};
};

}
```

### Type erasure with static free template functions and SSO

- SSO = small size optimization

- The trick is that a function pointer can point to functions with different template arguments
``` c++
template <typename T>
void templated_copy(void* storage) { /* ... */ }

using copy_fp_t = void (*)(void* storage);

// the same `function pointer` can point to multiple templates
copy_fp_t copy = templated_copy<int>;
copy = templated_copy<double>;
```

- main idea : type erasure = store the data in a buffer + function pointers 
	- store the data in a buffer (placement new for SSO or store the pointer to heap allocated)
	- mimic a virtual table to interpret the stored data according to the real type

``` c++
namespace acpp {
namespace detail {

struct _Operations;

// Raw bytes to store the callable.
// a) in-place constructed callable if it fits, or 
// b) pointer to the heap allocated callable
struct _Callable_storage {
	uint64_t mem1;
	uint64_t mem2;
};

// The callable storage + the virtual table function pointers
struct _Any_callable {
	_Callable_storage storage;
	_Operations* operations{nullptr};
};

// Concept for callables that can be stored in the local buffer to avoid dynamic memory allocations
template <typename T>
concept _In_place_callable = 
	   sizeof(T) <= sizeof(_Any_callable::storage) 
	&& std::alignment_of<_Callable_storage>::value % 
			std::alignment_of<T>::value == 0 
	&& std::is_nothrow_move_constructible<T>::value;

// Handle the callables that can't be stored in the local buffer
template <typename LargeCallable>
struct _Any_callable_manager {

	static LargeCallable*& get_ptr(const _Any_callable& any_callable) noexcept {
		return *const_cast<LargeCallable**>(
		reinterpret_cast<const LargeCallable* const*>(&any_callable.storage));
	}

	static LargeCallable& get_ref(const _Any_callable& any_callable) noexcept {
		return *get_ptr(any_callable);
	}

	template <typename R, typename... Args>
		static R invoke(const _Any_callable& any_callable, Args... args) {
		return get_ref(any_callable)(std::forward<Args>(args)...);
	}

	template <typename _Fn>
		static void store(_Fn&& callable, _Any_callable& any_callable) {
		get_ptr(any_callable) = new LargeCallable(std::forward<_Fn>(callable));
	}

	static void move_and_destroy(_Any_callable& dest, 
						         _Any_callable& src) noexcept {
		dest.storage = src.storage;
		dest.operations = std::exchange(src.operations, nullptr);
	}

	static void destroy(_Any_callable& any_callable) {
		delete get_ptr(any_callable);
	}
};

// Handle the callables that can be stored in the local buffer
template <typename Callable> requires _In_place_callable<Callable>
struct _Any_callable_manager<Callable> {

	static Callable& get_ref(const _Any_callable& any_callable) noexcept {
		return const_cast<Callable&>(
		reinterpret_cast<const Callable&>(any_callable.storage));
	}

	template <typename R, typename... Args>
		static R invoke(const _Any_callable& any_callable, Args... args) {
		return get_ref(any_callable)(std::forward<Args>(args)...);
	}

	template <typename _Fn>
	static void store(_Fn&& callable, _Any_callable& any_callable) {
		new (&get_ref(any_callable)) Callable(std::forward<_Fn>(callable));
	}

	static void move_and_destroy(_Any_callable& dest, _Any_callable& src) {
		store(std::move(get_ref(src)), dest);
		destroy(src);
		dest.operations = std::exchange(src.operations, nullptr);
	}

	static void destroy(_Any_callable& any_callable) {
		get_ref(any_callable).~Callable();
	}
};

/// Acts like a virtual table
struct _Operations {
	template <typename Callable>
	static _Operations& create_operations() {
		static _Operations vtable;
		vtable.destroy = &templated_destroy<Callable>;
		vtable.copy = &templated_copy<Callable>;
		vtable.move = &templated_move<Callable>;
		return vtable;
	}

	template <typename Callable>
	static void templated_destroy(_Any_callable& any_callable) {
		_Any_callable_manager<Callable>::destroy(any_callable);
	}

	template <typename Callable>
	static void templated_copy(_Any_callable& dst, const _Any_callable& src) {
		using _Manager = _Any_callable_manager<Callable>;
		_Manager::store(_Manager::get_ref(src), dst);
		dst.operations = src.operations;
	}

	template <typename Callable>
	static void templated_move(_Any_callable& dst, _Any_callable& src) {
		_Any_callable_manager<Callable>::move_and_destroy(dst, src);
	}

	void (*destroy)(_Any_callable& any_callable);
	void (*copy)(_Any_callable& dst, const _Any_callable& src);
	void (*move)(_Any_callable& dst, _Any_callable& src);
};

} // namespace detail

template <typename... Args>
class function;

template <typename Callable, typename R, typename... Args>
concept _Is_valid_callable =
	// Callable to match the signature
	std::same_as<std::invoke_result_t<Callable, Args...>, R> &&
	// Different than function to distinguish from the copy constructor
	!std::is_same_v<std::remove_cvref_t<Callable>, function<R(Args...)>>;

template <typename R, typename... Args>
class function<R(Args...)> {
private:
	using _Callable_invoker = R (*)(const detail::_Any_callable&, Args...);

public:
	function() noexcept = default;
	function(const function& oth) {
		if (oth) {
			oth.any_callable_.operations->copy(any_callable_, oth.any_callable_);
			invoker_ = oth.invoker_;
		}
	}
	function(function&& oth) noexcept {
		if (oth) {
			oth.any_callable_.operations->move(any_callable_, oth.any_callable_);
			invoker_ = std::exchange(oth.invoker_, nullptr);
		}
	}

	// Copy and swap idiom
	function& operator=(function oth) {
		swap(oth);
		return *this;
	}

	template <typename Callable> 
	requires _Is_valid_callable<Callable, R, Args...>
	function(Callable&& callable) { set(std::forward<Callable>(callable)); }

	template <typename Callable>
	function& operator=(Callable&& callable) {
		if (*this) { unset(); }
		set(std::forward<Callable>(callable));
	}

	~function() { if (*this) { unset(); } }
	
	R operator()(Args... args) {
		if (!any_callable_.operations) {
			throw std::runtime_error("bad function call");
		}
		return (*invoker_)(any_callable_, std::forward<Args>(args)...);
	}

	operator bool() const noexcept {
		return any_callable_.operations != nullptr;
	}

	void swap(function& oth) noexcept {
		detail::_Any_callable temp_callable;
		if (oth) { 
			oth.any_callable_.operations->move(temp_callable, 
										       oth.any_callable_);
		}
		if (*this) { 
			any_callable_.operations->move(oth.any_callable_, 
										   any_callable_); 
		}
		if (temp_callable.operations) { 
			temp_callable.operations->move(any_callable_, temp_callable); 
		}
		std::swap(invoker_, oth.invoker_);
}

private:
	template <typename Callable>
	void set(Callable&& callable) {
		using _CleanCallable = std::decay_t<Callable>;
		using _Manager = detail::_Any_callable_manager<_CleanCallable>;
		_Manager::store(std::forward<Callable>(callable), any_callable_);
		any_callable_.operations =
			 &detail::_Operations::create_operations<_CleanCallable>();
		invoker_ = &_Manager::template invoke<R, Args...>;
	}

	void unset() {
		any_callable_.operations->destroy(any_callable_);
		any_callable_.operations = nullptr;
	}

private:
	// Local storage + Vtable to interpret data
	detail::_Any_callable any_callable_;
	// Separate function pointer for invoker to avoid the overhead of 
	// double indirection (vtable + func-pointer)
	_Callable_invoker invoker_{nullptr};
};

}

```

### GCC implementation of std::function 

- https://github.com/gcc-mirror/gcc/blob/master/libstdc++-v3/include/bits/std_function.h

### SSO implementation of std::function 

- https://github.com/skarupke/std_function/blob/master/function.h#L52
- https://probablydance.com/2013/01/13/a-faster-implementation-of-stdfunction/