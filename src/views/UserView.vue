<template>
  <div>
    <v-container fluid class="pa-4">
      <TableTop
        :search="search"
        @updateSearch="updateSearch"
        @open-modal="openModalCreate"
        :PageTitle="PageTitle"
      />
      <v-row wrap class="table">
        <v-data-table
          dark
          :loading="loadingTable"
          :headers="headers"
          :header-props="headerprops"
          mobile-breakpoint="890"
          :items="users"
          :items-per-page="7"
          class="elevation-1"
          item-key="id"
          :search="search"
          :custom-filter="filter"
          :no-results-text="noDataText"
          :footer-props="{
            'items-per-page-text': 'Registros por página',
            'items-per-page-options': [7, 10, 15, this.users.length],
          }"
        >
          <template v-slot:[`item.acoes`]="{ item }">
            <td>
              <v-icon class="mr-2" @click="openModalEdit(item)">
                mdi-pencil
              </v-icon>
              <v-icon class="mr-2" @click="openModalDelete(item)">
                mdi-delete
              </v-icon>
            </td>
          </template>
        </v-data-table>
      </v-row>
    </v-container>
    <!-- modal -->
    <v-row justify="center">
      <v-dialog v-model="dialog" persistent max-width="500px">
        <v-form ref form>
          <v-card dark>
            <v-card-title>
              <span class="text-h5">{{ ModalTitle }}</span>
            </v-card-title>
            <v-card-text>
              <v-col>
                <v-col cols="12">
                  <v-text-field
                    v-model="nome"
                    label="Nome"
                    required
                    :error-messages="NameError"
                    @input="$v.nome.$touch()"
                    @blur="$v.nome.$touch()"
                  ></v-text-field>
                </v-col>
                <v-col cols="12">
                  <v-text-field
                    v-model="cidade"
                    label="Cidade"
                    required
                    :error-messages="CityError"
                    @input="$v.cidade.$touch()"
                    @blur="$v.cidade.$touch()"
                  ></v-text-field>
                </v-col>
                <v-col cols="12">
                  <v-text-field
                    v-model="endereco"
                    label="Endereço"
                    required
                    :error-messages="AddressError"
                    @input="$v.endereco.$touch()"
                    @blur="$v.endereco.$touch()"
                  ></v-text-field>
                </v-col>
                <v-col cols="12">
                  <v-text-field
                    v-model="email"
                    :counter="50"
                    label="Email"
                    type="email"
                    required
                    :error-messages="EmailError"
                    @input="validateEmail()"
                    @blur="$v.email.$touch()"
                  ></v-text-field>
                </v-col>
              </v-col>
            </v-card-text>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn color="red darken-1" text @click="closeModal">
                Cancelar
              </v-btn>
              <v-btn color="blue darken-1" text @click="confirm">
                Salvar
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-form>
      </v-dialog>
    </v-row>
    <!-- modal delete -->
    <v-row justify="center">
      <v-dialog v-model="dialogDelete" persistent max-width="600px">
        <v-card dark>
          <v-card-title>
            <span class="text-h5">Excluir Usuário</span>
          </v-card-title>
          <v-card-text>
            Tem certeza que deseja excluir o usuário selecionado?
          </v-card-text>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="red darken-1" text @click="closeModalDelete">
              Cancelar
            </v-btn>
            <v-btn color="green darken-1" text @click="confirmDelete">
              Confirmar
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </v-row>
  </div>
</template>

<script>
import User from "@/services/users";
import Swal from "sweetalert2";
import { validationMixin } from "vuelidate";
import { required, maxLength, email } from "vuelidate/lib/validators";
import TableTop from "@/components/TableTop";

export default {
  components: {
    TableTop,
  },

  mixins: [validationMixin],
  validations: {
    nome: { required },
    endereco: { required },
    cidade: { required },
    email: { required, email, maxLength: maxLength(50) },
  },
  data() {
    return {
      noDataText: "Nenhum registro encontrado",
      search: "",
      ModalTitle: "",
      PageTitle: "Usuários",
      headers: [
        { text: "ID", value: "id" },
        { text: "Nome", value: "nome" },
        { text: "Cidade", value: "cidade" },
        { text: "Endereço", value: "endereco" },
        { text: "Email", value: "email" },
        { text: "Ações", value: "acoes", sortable: false },
      ],
      headerprops: {
        sortByText: "Ordenar Por",
      },
      users: [],
      nome: "",
      endereco: "",
      cidade: "",
      email: "",
      dialog: false,
      dialogDelete: false,
      userId: null,
      emailExists: false,
      loadingTable: false,
    };
  },
  computed: {
    // validacao
    NameError() {
      const errors = [];
      if (!this.$v.nome.$dirty) return errors;
      !this.$v.nome.required && errors.push("Informe o nome.");
      return errors;
    },
    AddressError() {
      const errors = [];
      if (!this.$v.endereco.$dirty) return errors;
      !this.$v.endereco.required && errors.push("Informe o endereco.");
      return errors;
    },
    CityError() {
      const errors = [];
      if (!this.$v.cidade.$dirty) return errors;
      !this.$v.cidade.required && errors.push("Informe a cidade.");
      return errors;
    },
    EmailError() {
      const errors = [];
      if (!this.$v.email.$dirty) return errors;
      !this.$v.email.maxLength && errors.psuh("O limite é de 50 caracteres.");
      !this.$v.email.email && errors.push("Informe um e-mail válido.");
      !this.$v.email.required && errors.push("Informe o e-mail.");
      if (this.emailExists) {
        errors.push("Este e-mail já existe no sistema.");
      }
      return errors;
    },
  },
  mounted() {
    this.listUsers();
  },
  methods: {
    updateSearch(newSearchValue) {
      this.search = newSearchValue;
    },
    // search
    filter(value, search) {
      return (
        value != null &&
        search != null &&
        (typeof value === "string" || typeof value === "number") &&
        value.toString().toLowerCase().indexOf(search.toLowerCase()) !== -1
      );
    },
    // Listar
    async listUsers() {
      this.loadingTable = true;
      try {
        const [usersResponse] = await Promise.all([User.list()]);

        this.users = usersResponse.data.map((user) => ({
          id: user.id,
          nome: user.nome,
          cidade: user.cidade,
          endereco: user.endereco,
          email: user.email,
        }));
        // Ordem por id
        this.users.sort((a, b) => {
          if (a.id > b.id) {
            return 1;
          } else if (a.id < b.id) {
            return -1;
          } else {
            return 0;
          }
        });
      } catch (error) {
        console.error("Erro ao buscar informações:", error);
      } finally {
        this.loadingTable = false;
      }
    },
    // Emails já cadastrados
    CheckEmails() {
      return this.users.some((user) => user.email == this.email);
    },
    validateEmail() {
      this.emailExists = this.CheckEmails(this.email);
      if (this.emailExists) {
        this.$v.email.$touch();
      }
    },
    // Abrir o modal para adicionar
    openModalCreate() {
      this.ModalTitle = "Adicionar Usuário";
      this.dialog = true;
      this.$v.$reset();

      this.nome = "";
      this.endereco = "";
      this.cidade = "";
      this.email = "";
    },
    // Abrir o modal para editar
    openModalEdit(user) {
      this.ModalTitle = "Editar Usuário";
      this.dialog = true;
      this.$v.$reset();

      this.userId = user.id;
      this.nome = user.nome;
      this.endereco = user.endereco;
      this.cidade = user.cidade;
      this.email = user.email;
    },
    // Fechar modal
    closeModal() {
      this.dialog = false;
    },
    // confirm
    confirm() {
      if (!this.emailExists) {
        this.$v.$touch();
        if (!this.$v.$error) {
          // Identica qual modal foi ativado (Add)
          if (this.ModalTitle === "Adicionar Usuário") {
            const newuser = {
              nome: this.nome,
              endereco: this.endereco,
              cidade: this.cidade,
              email: this.email,
            };
            User.create(newuser)
              .then((response) => {
                this.users.push({ id: response.data.id, ...newuser });
                Swal.fire({
                  icon: "success",
                  title: "Usuário adicionado com êxito!",
                  showConfirmButton: false,
                  timer: 3500,
                });
                this.closeModal();
              })
              .catch((error) => {
                console.error("Erro ao adicionar o usuario:", error);
                Swal.fire({
                  icon: "error",
                  title: "Erro ao adicionar usuário.",
                  text: error.response.data.error,
                  showConfirmButton: false,
                  timer: 3500,
                });
              });
          }
          // Caso contrário, edita
          else {
            const editeduser = {
              id: this.userId,
              nome: this.nome,
              endereco: this.endereco,
              cidade: this.cidade,
              email: this.email,
            };
            User.update(editeduser)
              .then(() => {
                this.users = this.users.map((user) => {
                  if (user.id === editeduser.id) {
                    return editeduser;
                  } else {
                    return user;
                  }
                });
                this.dialogEdit = false;
                Swal.fire({
                  icon: "success",
                  title: "Usuário atualizado com êxito!",
                  showConfirmButton: false,
                  timer: 3500,
                });
                this.closeModal();
              })
              .catch((error) => {
                console.error("Erro ao atualizar usuário:", error);
                Swal.fire({
                  icon: "success",
                  title: "Erro ao atualizar usuário.",
                  text: error.response.data.error,
                  showConfirmButton: false,
                  timer: 3500,
                });
                this.closeModal();
              });
          }
        }
      }
    },
    // Excluir
    openModalDelete(user) {
      this.userId = user.id;
      this.nome = user.nome;
      this.endereco = user.endereco;
      this.cidade = user.cidade;
      this.email = user.email;
      this.dialogDelete = true;
    },
    closeModalDelete() {
      this.dialogDelete = false;
    },
    confirmDelete() {
      const deleteduser = {
        id: this.userId,
        nome: this.nome,
        endereco: this.endereco,
        cidade: this.cidade,
        email: this.email,
      };
      User.delete(deleteduser)
        .then((response) => {
          if (response.status === 200) {
            Swal.fire({
              icon: "success",
              title: "Usuário deletado com êxito!",
              showConfirmButton: false,
              timer: 3500,
            });
            this.listUsers();
            this.closeModalDelete();
          } else {
            Swal.fire({
              icon: "error",
              title: "Erro ao deletar usuário.",
              showConfirmButton: false,
              timer: 3500,
            });
          }
        })
        .catch((e) => {
          console.error("Erro ao deletar a usuário:", e);
          Swal.fire({
            icon: "error",
            title: "Erro ao deletar usuário.",
            text: e.response.data.error,
            showConfirmButton: false,
            timer: 3500,
          });
          this.closeModalDelete();
        });
    },
  },
};
</script>
