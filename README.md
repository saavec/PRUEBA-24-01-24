import pg8000
import pandas as pd
import plotly.express as px
import dash
from dash import dcc, html
from dash.dependencies import Input, Output

# Detalles de conexión a la base de datos
host = 'localhost'
port = 5432
database = 'postgres'
user = 'postgres'
password = 'Cardenas96*'

# Crea la conexión
with pg8000.connect(host=host, port=port, database=database, user=user, password=password) as conn:
    # Consulta SQL para obtener datos
    consulta_sql = '''
        SELECT p2."CODIGO", ce."empresa", p2."ENTREGA_GWH", p2."RETIRO_GWH", p2."FECHA"
        FROM principal2 p2
        INNER JOIN cambio_empresas ce ON p2."CODIGO" = ce."codigo"
    '''

    # Leer datos de la base de datos
    df = pd.read_sql(consulta_sql, conn)

# Convierte la columna 'FECHA' a datetime
df['FECHA'] = pd.to_datetime(df['FECHA'])

# Inicia la aplicación Dash
app = dash.Dash(__name__)

# Diseño de la interfaz de usuario
app.layout = html.Div([
    dcc.Dropdown(
        id='year-month-dropdown',
        options=[
            {'label': f"{year}-{month:02d}", 'value': f"{year}-{month:02d}"}
            for year in df['FECHA'].dt.year.unique()
            for month in range(1, 13)
        ],
        value=f"{df['FECHA'].dt.year.max()}-{df['FECHA'].dt.month.max():02d}",
        multi=False,
        style={'width': '40%'}
    ),
    dcc.Dropdown(
        id='order-dropdown',
        options=[
            {'label': 'ENTREGA_GWH', 'value': 'ENTREGA_GWH'},
            {'label': 'RETIRO_GWH', 'value': 'RETIRO_GWH'},
            {'label': 'E-R', 'value': 'E-R'}
        ],
        value='ENTREGA_GWH',
        multi=False,
        style={'width': '40%'}
    ),
    dcc.Input(
        id='filter-input',
        type='text',
        placeholder='Ingrese el valor',
        style={'width': '10%', 'margin-top': '10px'}  # Ajusta según sea necesario
    ),
    dcc.Graph(
        id='bar-chart',
        style={'height': '700px', 'width': '1000px'}  # Ajusta según sea necesario
    )
])

# Callback para actualizar el gráfico según la fecha, el filtro de orden y los valores excluidos
@app.callback(
    Output('bar-chart', 'figure'),
    [Input('year-month-dropdown', 'value'),
     Input('order-dropdown', 'value'),
     Input('filter-input', 'value')]
)
def update_chart(selected_year_month, order_value, filter_value):
    selected_year, selected_month = map(int, selected_year_month.split('-'))
    filtered_df = df[(df['FECHA'].dt.year == selected_year) & (df['FECHA'].dt.month == selected_month)]

    # Filtrar valores menores o iguales al límite ingresado
    if filter_value and filter_value.isdigit():
        limit_value = int(filter_value)
        filtered_df = filtered_df[(filtered_df['ENTREGA_GWH'] > limit_value) & (filtered_df['RETIRO_GWH'] > limit_value)]

    # Aplica el filtro de ordenamiento a toda la gráfica de barras
    if order_value == 'E-R':
        filtered_df['E-R'] = filtered_df['ENTREGA_GWH'] - filtered_df['RETIRO_GWH']
        filtered_df = filtered_df.sort_values(by=['E-R'], ascending=False)
    else:
        filtered_df = filtered_df.sort_values(by=[order_value], ascending=False)

    fig = px.bar(filtered_df, x='empresa', y=['ENTREGA_GWH', 'RETIRO_GWH'],
                 labels={'value': 'GWH'},
                 title=f'Gráfico de Barras Agrupadas - {selected_year}-{selected_month:02d}',
                 barmode='group')

    fig.update_layout(xaxis_title='empresa')

    return fig

# Ejecutar la aplicación en el modo de depuración
if __name__ == '__main__':
    try:
        app.run_server(debug=True, use_reloader=False, port=8052)
    except OSError as e:
        if "Address 'http://127.0.0.1:8052' already in use" in str(e):
            print("El puerto 8052 ya está en uso. Cierra la aplicación que lo está utilizando o cambia el puerto.")
        else:
            raise e
