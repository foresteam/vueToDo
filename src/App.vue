<template>
	<img alt="Vue logo" src="./assets/logo.png">
	<TodoAdd @add-todo="addTodo" />
	<PajiloyFilter :filterNames="filterNames" @filter-change="applyFilter" />
	<Loading v-if="loading" />
	<TodoList v-else-if="todos.length > 0" :todos="todos" :filter="filter" @remove-todo="removeTodo" @update-completed="updateCompleted" />
	<div v-else>No todos!</div>
</template>

<script>
import TodoList from '@/components/TodoList';
import TodoAdd from '@/components/TodoAdd';
import PajiloyFilter from '@/components/PajiloyFilter';
import Loading from '@/components/Loading';

const proto = 'http';
const server = '://dfmfd.ddns.net:1337/';

const xhttp = ({url, method, params}) => new Promise((resolve, reject) => { // my ajax replacement. Mb i didn't actually need this. Idk
	method ||= 'POST';
	params ||= {};

	let xhr = new XMLHttpRequest();
	xhr.open(method, url);
	xhr.onload = event => {
		resolve(JSON.parse(xhr.responseText || '{}'), event)
	};
	xhr.onerror = reject;
	xhr.setRequestHeader("Content-Type", "application/json;charset=UTF-8");
	xhr.send(JSON.stringify(params));
});

function ToDo(text, id, completed) { // weird shit. Don't think it's needed now, but i'm too lazy to remove it
	this.text = text;
	this.id = id;
	this.completed = completed || false;
}

let ws; // declaring websocket

export default {
	name: 'App',
	data: () => ({
		todos: [
			new ToDo('Kill urself', 0)
		],
		filterNames: [
			'All',
			'Completed',
			'Not completed'
		],
		filter: 'All',
		loading: false
	}),
	methods: {
		addTodo(text) {
			let lastId = (this.todos[this.todos.length - 1] || {}).id + 1 || 0;
			let ntodo = new ToDo(text, lastId);
			xhttp({
				url: proto + server + 'actions/addTodo',
				params: {
					ntodo: { text: ntodo.text, id: ntodo.id, completed: ntodo.completed }
				}
			});
		},
		removeTodo(id) {
			xhttp({ url: proto + server + 'actions/removeTodo', params: { id } });
		},
		updateCompleted(todo, nv) {
			xhttp({ url: proto + server + 'actions/updateTodo', params: { id: todo.id, completed: nv } });
		},
		applyFilter(value) {
			this.filter = value;
		},
		// ws_ methods are to be called by the server through ws. They apply the changes we (or smb else) make
		ws_loadTodos(todos) {
			this.todos = todos.map(e => new ToDo(...Object.values(e)));
		},
		ws_addTodo(todo) {
			this.todos.push(new ToDo(...Object.values(todo)));
		},
		ws_removeTodo(id) {
			this.todos.splice(this.todos.findIndex(e => e.id == id), 1);
		},
		ws_updateTodo({id, completed}) {
			this.todos.find(e => e.id == id).completed = completed;
		}
	},
	components: {
		TodoList,
		TodoAdd,
		PajiloyFilter,
		Loading
	},
	async mounted() {
		// i thought sockets would be better as they add "cool dynamics"
		// this.todos = (await xhttp({ url: proto + server + 'actions/getTodos' })).map(e => new ToDo(...Object.values(e)));

		ws = new WebSocket('ws' + server + ''); // there could be a path
		//ws.onopen = () => ws.send('Fuck y!'); // very important line
		ws.addEventListener('message', ({data: msg}) => {
			this.loading = true;
			let json = JSON.parse(msg);
			this['ws_' + json.action](json.data);
			this.loading = false;
		});
	}
}
</script>

<style>
#app {
	font-family: Avenir, Helvetica, Arial, sans-serif;
	-webkit-font-smoothing: antialiased;
	-moz-osx-font-smoothing: grayscale;
	text-align: center;
	color: #2c3e50;

	margin: auto;
	width: 70%;

	margin-top: 60px;
}
</style>
