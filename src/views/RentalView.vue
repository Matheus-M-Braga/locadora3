<template>
  <div class="container_master">
    <v-container>
      <v-layout row wrap justify-space-between class="top">
        <div class="title_btn">
          <h1 class="subheading black--text">Aluguéis</h1>
          <v-btn dark @click="openModalCreate">Novo +</v-btn>
        </div>
        <div class="searchbar">
          <input type="text" v-model="search" placeholder="Pesquisar..." />
          <v-icon>mdi-magnify</v-icon>
        </div>
      </v-layout>
      <v-layout row wrap class="table">
        <v-data-table
          dark
          :headers="headers"
          :items="filteredRentals"
          :items-per-page="7"
          mobile-breakpoint="890"
          class="elevation-1"
          :no-results-text="noDataText"
          :header-props="headerprops"
          :footer-props="{
            'items-per-page-text': 'Registros por página',
            'items-per-page-options': [7, 10, 15, this.rentals.length],
          }"
        >
          <template v-slot:[`item.status`]="{ item }">
            <td>
              <v-chip :class="statusClass(item)" class="black--text">
                {{ item.status }}
              </v-chip>
            </td>
          </template>
          <template v-slot:[`item.acoes`]="{ item }">
            <td>
              <v-icon class="mr-2" @click="openModalDevol(item)">
                mdi-book
              </v-icon>
              <v-icon class="" @click="openModalDelete(item)">
                mdi-delete
              </v-icon>
            </td>
          </template>
        </v-data-table>
      </v-layout>
    </v-container>
    <!-- modal -->
    <v-row justify="center">
      <v-dialog v-model="dialog" persistent max-width="500px">
        <v-card dark>
          <v-card-title>
            <span class="text-h5">{{ ModalTitle }}</span>
          </v-card-title>
          <v-card-text>
            <v-col>
              <v-col cols="12">
                <v-select
                  v-model="livro_id"
                  :items="listBooks"
                  item-text="nome"
                  label="Livro"
                  required
                  :error-messages="BookError"
                  @input="$v.livro_id.$touch()"
                  @blur="$v.livro_id.$touch()"
                ></v-select>
              </v-col>
              <v-col cols="12">
                <v-select
                  v-model="usuario_id"
                  :items="listUsers"
                  item-text="nome"
                  label="Usuário"
                  required
                  :error-messages="UserError"
                  @input="$v.usuario_id.$touch()"
                  @blur="$v.usuario_id.$touch()"
                ></v-select>
              </v-col>
              <v-col cols="12">
                <v-text-field
                  disabled
                  v-model="data_aluguel"
                  label="Data do Aluguel (Hoje)"
                  required
                  :error-messages="AlugError"
                  @input="$v.data_aluguel.$touch()"
                  @blur="$v.data_aluguel.$touch()"
                  type="date"
                ></v-text-field>
              </v-col>
              <v-col cols="12">
                <v-text-field
                  v-model="data_previsao"
                  label="Previsão de Devolução"
                  required
                  :error-messages="DevolError"
                  @input="$v.data_previsao.$touch()"
                  @blur="$v.data_previsao.$touch()"
                  type="date"
                  :min="data_aluguel"
                  :max="MaxDate()"
                ></v-text-field>
              </v-col>
            </v-col>
          </v-card-text>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="red darken-1" text @click="closeModal">
              Cancelar
            </v-btn>
            <v-btn color="blue darken-1" text @click="confirm"> Salvar </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </v-row>
    <!-- modal delete -->
    <v-row justify="center">
      <v-dialog v-model="dialogDelete" persistent max-width="600px">
        <v-card dark>
          <v-card-title>
            <span class="text-h5">Excluir Aluguel</span>
          </v-card-title>
          <v-card-text>
            <v-container>
              Tem certeza que deseja excluir o aluguel selecionado?
            </v-container>
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
    <!-- modal devol -->
    <v-row justify="center">
      <v-dialog v-model="dialogDevol" persistent max-width="600px">
        <v-card dark>
          <v-card-title>
            <span class="text-h5">Devolver Livro</span>
          </v-card-title>
          <v-card-text>
            <v-container>
              Tem certeza que deseja devolver o livro?
            </v-container>
          </v-card-text>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="red darken-1" text @click="closeModalDevol">
              Cancelar
            </v-btn>
            <v-btn color="green darken-1" text @click="confirmDevol">
              Confirmar
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </v-row>
  </div>
</template>

<script>
import Rental from "@/services/rental";
import Book from "@/services/book";
import User from "@/services/users";
import Swal from "sweetalert2";
import { validationMixin } from "vuelidate";
import { required } from "vuelidate/lib/validators";
export default {
  mixins: [validationMixin],

  validations: {
    livro_id: { required },
    usuario_id: { required },
    data_aluguel: { required },
    data_previsao: { required },
  },
  data() {
    return {
      noDataText: "Nenhum registro encontrado",
      search: "",
      ModalTitle: "",
      headers: [
        { text: "ID", value: "id" },
        { text: "Livro", align: "start", value: "livro_id" },
        { text: "Usuário", value: "usuario_id" },
        { text: "Data do Aluguel", value: "data_aluguel" },
        { text: "Previsão de Devolução", value: "data_previsao" },
        { text: "Data de Devolução", value: "data_devolucao" },
        { text: "Status", value: "status" },
        { text: "Ações", value: "acoes", sortable: false },
      ],
      headerprops: {
        sortByText: "Ordenar Por",
      },
      rentals: [],
      listBooks: [],
      listUsers: [],
      livro_id: "",
      usuario_id: "",
      data_aluguel: "",
      data_previsao: "",
      data_devolucao: "",
      status: "",
      dialog: false,
      dialogDelete: false,
      dialogDevol: false,
      rentalId: null,
    };
  },
  computed: {
    // validacao
    BookError() {
      const errors = [];
      if (!this.$v.livro_id.$dirty) return errors;
      !this.$v.livro_id.required && errors.push("Informe o livro.");
      return errors;
    },
    UserError() {
      const errors = [];
      if (!this.$v.usuario_id.$dirty) return errors;
      !this.$v.usuario_id.required && errors.push("Informe o usuário.");
      return errors;
    },
    AlugError() {
      const errors = [];
      if (!this.$v.data_aluguel.$dirty) return errors;
      !this.$v.data_aluguel.required &&
        errors.push("Informe a data do aluguel.");
      return errors;
    },
    DevolError() {
      const errors = [];
      if (!this.$v.data_previsao.$dirty) return errors;
      !this.$v.data_previsao.required &&
        errors.push("Informe a previsão de devolução.");
      return errors;
    },
    filteredRentals() {
      const searchValue = this.search.toLowerCase();
      return this.rentals.filter((rental) => {
        for (const prop in rental) {
          const propValue = rental[prop].toString().toLowerCase();
          if (propValue.includes(searchValue)) {
            return true;
          }
        }
        return false;
      });
    },
  },
  mounted() {
    this.listAlugs();
  },
  methods: {
    // Calcula a data limite
    MaxDate() {
      const today = new Date();
      const futureDate = new Date(today);
      futureDate.setDate(today.getDate() + 30);
      return futureDate.toISOString().substr(0, 10);
    },
    statusClass(item) {
      if (item.status == "Atrasado") {
        return "red";
      } else if (item.status == "No prazo") {
        return "green";
      } else {
        return "yellow";
      }
    },
    // Formatar datas
    formatDate(dateString) {
      const utcDate = new Date(dateString);
      const localDate = new Date(
        utcDate.getTime() + utcDate.getTimezoneOffset() * 60000
      );
      const options = {
        year: "numeric",
        month: "2-digit",
        day: "2-digit",
        timeZone: "UTC",
      };
      return localDate.toLocaleDateString("pt-BR", options);
    },
    // Converter para o padrão da API
    parseDate(date) {
      if (!date) return null;

      const [day, month, year] = date.split("/");
      return `${year}-${month}-${day}`;
    },
    // Listar
    async listAlugs() {
      try {
        const [booksResponse, rentalsResponse, usersResponse] =
          await Promise.all([Book.list(), Rental.list(), User.list()]);

        this.listBooks = booksResponse.data.map((livro) => ({
          id: livro.id,
          nome: livro.nome,
        }));

        this.listUsers = usersResponse.data.map((usuario) => ({
          id: usuario.id,
          nome: usuario.nome,
        }));

        this.rentals = rentalsResponse.data.map((rental) => {
          const devolucaoDate = rental.data_devolucao;
          const previsaoDate = rental.data_previsao;
          let statusInfo;
          if (devolucaoDate !== null) {
            if (devolucaoDate > previsaoDate) {
              statusInfo = "Atrasado";
            } else {
              statusInfo = "No prazo";
            }
          } else {
            statusInfo = "Pendente";
          }
          return {
            id: rental.id,
            livro_id: rental.livro_id.nome,
            usuario_id: rental.usuario_id.nome,
            data_aluguel: this.formatDate(rental.data_aluguel),
            data_previsao: this.formatDate(rental.data_previsao),
            data_devolucao: rental.data_devolucao
              ? this.formatDate(rental.data_devolucao)
              : "...",
            status: statusInfo,
          };
        });
        // Pendentes primeiro
        this.rentals.sort((a, b) => {
          if (a.status === "Pendente" && b.status !== "Pendente") {
            return -1;
          } else if (a.status !== "Pendente" && b.status === "Pendente") {
            return 1;
          } else {
            return 0;
          }
        });
      } catch (error) {
        console.error("Erro ao buscar informações:", error);
      }
    },
    // Abrir o modal para adicionar
    openModalCreate() {
      this.ModalTitle = "Adicionar Aluguel";
      this.dialog = true;
      this.$v.$reset();

      this.livro_id = "";
      this.usuario_id = "";
      this.data_aluguel = new Date().toISOString().substr(0, 10);
      this.data_previsao = "";
      this.data_devolucao = "";
    },
    // Fechar modal
    closeModal() {
      this.dialog = false;
    },
    // confirm
    confirm() {
      this.$v.$touch();
      if (!this.$v.$error) {
        // Identifica qual modal foi ativado (Add)
        if (this.ModalTitle === "Adicionar Aluguel") {
          const selectedBook = this.listBooks.find(
            (livro) => livro.nome === this.livro_id
          );
          const selectedUser = this.listUsers.find(
            (usuario) => usuario.nome === this.usuario_id
          );

          const novoAlug = {
            livro_id: selectedBook,
            usuario_id: selectedUser,
            data_aluguel: this.data_aluguel,
            data_previsao: this.data_previsao,
          };
          Rental.create(novoAlug)
            .then((response) => {
              this.rentals.push({ id: response.data.id, ...novoAlug });
              Swal.fire({
                icon: "success",
                title: "Aluguel adicionado com êxito!",
                showConfirmButton: false,
                timer: 3500,
              });
              this.closeModal();
              this.listAlugs();
            })
            .catch((error) => {
              console.error("Erro ao adicionar o aluguel:", error);
              Swal.fire({
                icon: "error",
                title: "Erro ao adicionar o aluguel: ",
                text: error.response.data.error,
                showConfirmButton: false,
                timer: 3500,
              });
            });
        }
      }
    },
    // Excluir
    openModalDelete(rental) {
      this.update = { ...rental };
      this.confirmDelete(this.update);
    },
    closeModalDelete() {
      this.dialogDelete = false;
    },
    confirmDelete(rental) {
      const selectedBook = this.listBooks.find(
        (book) => book.nome === rental.livro_id
      );
      const selectedUser = this.listUsers.find(
        (user) => user.nome === rental.usuario_id
      );
      const deleteAlug = {
        id: rental.id,
        livro_id: selectedBook,
        usuario_id: selectedUser,
        data_aluguel: this.parseDate(rental.data_aluguel),
        data_previsao: this.parseDate(rental.data_previsao),
        data_devolucao: rental.data_devolucao !== "..." ? rental.data_devolucao : null,
      };
      Rental.delete(deleteAlug)
        .then((response) => {
          if (response.status === 200) {
            Swal.fire({
              icon: "success",
              title: "Aluguel deletado com êxito!",
              showConfirmButton: false,
              timer: 3500,
            });
            this.removeRental(deleteAlug.id);
            this.closeModalDelete();
          } else {
            Swal.fire({
              icon: "error",
              title: "Erro ao deletar aluguel!",
              showConfirmButton: false,
              timer: 3500,
            });
          }
        })
        .catch((e) => {
          console.error("Erro ao deletar aluguel:", e);
          Swal.fire({
            icon: "error",
            title: "Erro ao deletar aluguel!",
            text: e.response.data.error,
            showConfirmButton: false,
            timer: 3500,
          });
          this.closeModalDelete();
        });
    },
    removeRental(rentalId) {
      this.rentals = this.rentals.filter((rental) => rental.id !== rentalId);
    },
    // Devolução
    openModalDevol(rental) {
      if (rental.data_devolucao == "...") {
        this.rentalId = rental.id;
        this.livro_id = rental.livro_id;
        this.usuario_id = rental.usuario_id;
        this.data_aluguel = this.parseDate(rental.data_aluguel);
        this.data_previsao = this.parseDate(rental.data_previsao);
        this.dialogDevol = true;
      } else {
        Swal.fire({
          icon: "error",
          title: "Livro já foi devolvido!",
          showConfirmButton: false,
          timer: 3500,
        });
        this.closeModalDevol();
      }
    },
    confirmDevol() {
      const selectedBook = this.listBooks.find(
        (livro) => livro.nome === this.livro_id
      );
      const selectedUser = this.listUsers.find(
        (usuario) => usuario.nome === this.usuario_id
      );

      const returnedRental = {
        id: this.rentalId,
        livro_id: selectedBook ? { ...selectedBook } : this.livro_id,
        usuario_id: selectedUser ? { ...selectedUser } : this.usuario_id,
        data_aluguel: this.data_aluguel,
        data_previsao: this.data_previsao,
        data_devolucao: new Date().toISOString().substr(0, 10),
      };
      Rental.update(returnedRental)
        .then(() => {
          this.rentals = this.rentals.map((rental) => {
            if (this.selectedALugId === returnedRental.id) {
              return returnedRental;
            } else {
              return rental;
            }
          });
          Swal.fire({
            icon: "success",
            title: "Devolução realizada com êxito!",
            showConfirmButton: false,
            timer: 3500,
          });
          this.closeModalDevol();
          this.listAlugs();
          this.removeRental(returnedRental.id);
        })
        .catch((error) => {
          console.error("Erro ao devolver aluguel:", error);
          Swal.fire({
            icon: "error",
            title: "Erro ao realizar devolução!",
            text: error.response.data.error,
            showConfirmButton: false,
            timer: 3500,
          });
        });
    },
    closeModalDevol() {
      this.dialogDevol = false;
    },
  },
};
</script>