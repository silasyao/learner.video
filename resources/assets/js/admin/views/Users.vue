<template>
<div class="row" style="margin-bottom:5px;">
    <div class="form-group">
        <div class="col-xs-12">
            <div class="btn-group">
                <button class="btn btn-default" @click="model = 'active'">活跃会员</button>
                <button class="btn btn-info" @click="model = 'notActive'">冻结会员</button>
                <button class="btn btn-danger" @click="model = 'trashed'" v-if="isBoss">回收站</button>
            </div>
        </div>
    </div>
</div>
<div class="row">
    <div class="col-xs-12">
        <div class="panel">

            <div class="alert alert-success" v-show="success">
                {{ message }}
            </div>

            <header class="panel-heading">
                会员列表
            </header>
            <div class="panel-body">
                <table class="table table-bordered">
                    <thead>
                        <tr>
                            <td v-for="column in columns">{{ column }}</td>
                            <td v-if="isBoss">冻结</td>
                            <td v-if="isBoss">删除</td>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="user in users">
                            <td>{{ user.id }}</td>
                            <td><img :src="user.avatar" width="20"></td>
                            <td>{{ user.username }}</td>
                            <td>{{ user.email }}</td>
                            <td>
                                <span v-for="role in user.roles"
                                    class="label label-{{ role.name }}">
                                    {{ role.display_name }}
                                </span>
                                <button class="btn btn-danger btn-xs pull-right"
                                    v-if="isBoss"
                                    @click="assignRole(user)">
                                    <i class="fa fa-user"></i>
                                </button>
                            </td>
                            <td>{{ user.created_at | date }}</td>
                            <td class="toggle_active" v-if="isBoss">
                                <input type="checkbox"
                                    id="is_active{{$index}}"
                                    @click="toggleUserActive(user)"
                                    :disabled="toggling"
                                    checked="{{ user.is_active }}"/>
                                <label for="is_active{{$index}}"
                                    ></label>
                            </td>
                            <td v-if="isBoss">
                                <button class="btn btn-danger btn-xs"
                                        v-if="removeAble"
                                        data-toggle="tooltip"
                                        title="移至回收站!"
                                        @click.stop="removeUser(user)">
                                    <i class="fa fa-trash"></i>
                                </button>
                                <button class="btn btn-danger btn-xs"
                                        v-if="deleteAble"
                                        data-toggle="tooltip"
                                        title="彻底删除!"
                                        @click.stop="deleteUser(user)">
                                    <i class="fa fa-close"></i>
                                </button>
                                <button class="btn btn-success btn-xs"
                                        v-if="deleteAble"
                                        data-toggle="tooltip"
                                        title="恢复用户!"
                                        @click.stop="restoreUser(user)">
                                    <i class="fa fa-plus-circle"></i>
                                </button>
                            </td>
                        </tr>
                    </tbody>
                </table>
                <div class="table-foot">
                    <ul class="pagination pagination-sm no-margin pull-right">
                        <p class="pagination__text">共 {{ total_page }} 页， 当前 {{ current_page }} 页</p>
                        <li>
                            <button class="btn"
                                @click.stop="paginate('first')"
                                :disabled="prev_page_url == null">首页</button>
                        </li>
                        <li>
                            <button class="btn"
                                @click.stop="paginate('prev')"
                                :disabled="prev_page_url == null">«</button>
                        </li>
                        <li>
                            <button class="btn"
                                @click.stop="paginate('next')"
                                :disabled="next_page_url == null">»</button>
                        </li>
                        <li>
                            <button class="btn"
                                @click.stop="paginate('last')"
                                :disabled="next_page_url == null">尾页</button>
                        </li>
                    </ul>
                </div>
            </div><!-- /.panel-body -->
        </div><!-- /.panel -->
    </div>
</div>

<form @submit.prevent="saveRoleToUser" id="saveRoleToUser">
    <div class="modal fade" id="assignRoleModal">
         <div class="modal-dialog modal-sm">
            <div class="modal-content">
                <div class="modal-header">
                    <button type="button" class="close" data-dismiss="modal" aria-hidden="true">&times;</button>
                    <h4 class="modal-title">选择角色</h4>
                </div>
                <div class="modal-body">
                    <div class="alert alert-danger" v-if="hasError">
                        {{ error }}
                    </div>
                    <!-- Title field -->
                    <div class="form-group">
                        <div class="checkbox" v-for="role in allRoles">
                            <label>
                                <input type="checkbox"
                                    v-model="saveRoles"
                                    value="{{ role.id }}">
                                {{ role.display_name }}
                            </label>
                        </div>
                    </div>
                    <div class="form-group">
                        <input type="submit"
                            value="确定"
                            class="btn btn-success form-control">
                    </div>
                </div>
            </div>
        </div>
    </div>
</form>
</template>

<script>
module.exports = {
    data() {
        return {
            columns: [
                'ID',
                '图像',
                '用户名',
                '邮箱',
                '身份',
                '注册时间'
            ],
            users: [],
            total: 0,
            current_page: 1,
            last_page: 1,
            total_page: 0,
            first_page_url: '/admin/users',
            prev_page_url: '',
            next_page_url: '',
            last_page_url: '',

            success: false,
            message: '',
            hasError: false,
            error: '',

            model: 'active',
            toggling: false,

            allRoles: [], // all roles from db.
            saveRoles: [], // roles form form.
            user: {} // to assign user.
        }
    },

    props: ['roles'],

    created() {
        this.getData(this.first_page_url);

        this.getAllRoles();
    },

    computed: {
        removeAble() {
            return this.model !== 'trashed';
        },
        deleteAble() {
            return this.model === 'trashed';
        },
        isBoss() {
            let names = this.roles.map(role => role.name);

            return names.indexOf('boss') != -1;
        }
    },

    watch: {
        'model': function(now, old) {
            switch (now) {
                case old:
                    break;
                case 'active':
                    this.first_page_url = '/admin/users';
                    this.getData(this.first_page_url);
                    break;
                case 'notActive':
                    this.first_page_url = '/admin/users/notActive';
                    this.getData(this.first_page_url);
                    break;
                case 'trashed':
                    this.first_page_url = '/admin/users/trashed';
                    this.getData(this.first_page_url);
                    break;
            }
        }
    },

    methods: {
        getAllRoles() {
            this.$http.get('/admin/roles')
                .then(response => {
                    this.allRoles = response.data.roles;
                });
        },

        getData(url) {
            this.$http.get(url).then(response => {
                let data = response.data;

                this.users = data.data;
                this.total = data.total;
                this.last_page = data.last_page;
                this.current_page = data.current_page;
                this.total_page = this.getTotalPage(data.total, data.per_page);
                this.prev_page_url = data.prev_page_url;
                this.next_page_url = data.next_page_url;
                this.last_page_url = this.first_page_url + '?page=' + this.total_page;
            });
        },

        getTotalPage(total, per_page) {
            return Math.ceil(total / per_page);
        },

        paginate(direction) {
            switch (direction) {
                case 'prev':
                    this.getData(this.prev_page_url);
                    this.current_page --;
                    break;
                case 'next':
                    this.getData(this.next_page_url);
                    this.current_page ++;
                    break;
                case 'first':
                    this.getData(this.first_page_url);
                    this.current_page = 1;
                    break;
                case 'last':
                    this.getData(this.last_page_url);
                    this.current_page = this.last_page;
                    break;
            }
        },

        removeUser(user) {
            this.$http.delete('/admin/users/remove/' + user.id)
                .then(function(response) {
                    this.removeUserFromUserList(user);

                    this.showMessage(response.data.message);
                });
        },

        deleteUser(user) {
            this.$http.delete('/admin/users/delete/' + user.id)
                .then(function(response) {
                    this.removeUserFromUserList(user);

                    this.showMessage(response.data.message);
                });
        },

        restoreUser(user) {
            this.$http.put('/admin/users/restore/' + user.id)
                .then(function(response) {
                    this.removeUserFromUserList(user);

                    this.showMessage(response.data.message);
                });
        },

        toggleUserActive(user) {
            if (this.toggling) {
                this.showMessage('正在修改用户状态');
            }

            this.toggling = true;

            this.$http.put('/admin/users/toggleActive/' + user.id)
                .then(function(response) {
                    this.showMessage(response.data.message);
                    this.toggling = false;
                    // this.removeUserFromUserList(user);
                });
        },

        showMessage(message) {
            this.success = true;
            this.message = message;

            setTimeout(function() {
                this.success = false;
            }.bind(this), 2800);
        },

        removeUserFromUserList(user) {
            this.users.$remove(user);
        },

        assignRole(user) {
            jQuery('#assignRoleModal').modal('show');
            this.user = user;

            this.saveRoles = user.roles.map(role => '' + role.id);
        },

        saveRoleToUser() {
            this.$http.post('/admin/roles/user/' + this.user.id, {
                roles: this.saveRoles
            })
            .then(response => {
                let data = response.data;
                let index = this.findIndexById(data.user.id);

                this.showMessage(data.message);

                this.users.$set(index, data.user);

                jQuery('#assignRoleModal').modal('hide');
            })
            .catch(response => {
                this.showError(response.data.error);
            });
        },

        findIndexById(id) {
            var ids = this.users.map(u => u.id);

            return ids.indexOf(parseInt(id));
        },

        showError(error) {
            this.hasError = true;
            this.error = error;

            setTimeout(() => {
                this.hasError = false;
            }, 2000);
        }
    }
}
</script>

<style>
.label-boss {
    background-color: #7dd999;
}
.label-user {
    background-color: #dcbeaa;
}
.label-admin {
    background-color: #65643a;
}
.pagination__text {
    text-align: right;
}
.modal>.modal-dialog {
    padding-top: 50px;
}
.toggle_active input {
    position: absolute;
    left: -9999px;
}
.toggle_active label {
    display: block;
    position: relative;
    padding: 10px;
    border-radius: 50%;
    width: 20px;
    margin: 0;
    background-color: #ccc;
    box-shadow: 0 0 8px rgba(0,0,0,.2);
    white-space: nowrap;
    cursor: pointer;
    user-select: none;
    transition: background .2s, box-shadow .2s;
}
.toggle_active label::before {
    content: "";
    display: block;
    position: absolute;
    top: 0;
    left: 0;
    border: 3px solid #fff;
    border-radius: 50%;
    height: 20px;
    width: 20px;
    transiton: background-color .2s;
}
.toggle_active label:hover,
.toggle_active input:focus + .toggle_active label {
    box-shadow: 0 0 8px rgba(0,0,0,.6);
}
.toggle_active input:checked + .toggle_active label {
    background-color: #FFF;
}
.toggle_active input:checked + label::before {
    background-color: #7dd999;
}
</style>
